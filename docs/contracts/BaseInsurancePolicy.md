---
sidebar_position: 4
---

# Base Insurance Policy Contract

- This page contains the documentation for the Base Insurance Policy contract. This contract is the main parent contract that is used to derive the child contracts. It contains the basic functionalities that are common to all the child contracts.



```js
/**
note This contract is the base contract for all the insurance policies. 
The functions can be overidden in the derived contracts to achieve desired functionality.
*/

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

import "./SharedData.sol";
import "@chainlink/contracts/src/v0.8/AutomationCompatible.sol";
import "@chainlink/contracts/src/v0.8/ChainlinkClient.sol";
import "@chainlink/contracts/src/v0.8/ConfirmedOwner.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

abstract contract BaseInsurancePolicy is AutomationCompatible, ChainlinkClient, ConfirmedOwner {
    using Chainlink for Chainlink.Request;
    /**
    note Events related to BaseInsurancePolicy contract
    */
    event TerminatePolicy(address indexed policyAddress, uint256 indexed timestamp);
    event PolicyRevived(address indexed policyAddress, uint256 indexed timestamp);
    event PolicyFunded(address indexed policyAddress, uint256 indexed timestamp);
    event PolicyClaimed(
        address indexed policyAddress,
        uint256 indexed timestamp,
        bool indexed claimed
    );
    event PolicyMatured(address indexed policyAddress, uint256 indexed timestamp);
    event PolicyWithdraw(
        address indexed policyAddress,
        uint256 indexed timestamp,
        uint256 withdrawnAmount
    );

    /**
    note Internal and private variables
     */

    uint256 private fee;
    AggregatorV3Interface internal s_priceFeed;
    SharedData.Policy internal s_policy;
    uint256 private s_lastPaymentTimestamp;
    uint256 private s_startTimestamp;
    uint256 private s_timePassedSinceCreation;
    address[] private s_admins;

    /**
    note Errors and Warnings related to BaseInsurancePolicy contract
     */

    error OnlyAdminAllowed();
    error OnlyManagerAllowed();
    error NotAllowedToWithdraw();
    error PolicyNotActive();
    error PolicyTerminated();
    error PolicyNotClaimable();
    error PolicyAlreadyClaimed();
    error PolicyNotRevivable();
    error PolicyNotFunded();
    error PolicyNotMatured();
    error PolicyNotInGracePeriod();
    error RevivalAmountNotCorrect();
    error PremiumAmountNotCorrect();
    error PolicyActive();
    error InsufficientBalance(uint256 contractBalance, uint256 amount);

    /**
    @notice modifiers for the contract
     */

    // Only admins can call certain functions
    modifier onlyAdmin() {
        bool allowed = false;
        for (uint8 i = 0; i < s_admins.length; i++) {
            if (msg.sender == s_admins[i]) {
                allowed = true;
                break;
            }
        }
        if (!allowed) revert OnlyAdminAllowed();
        _;
    }

    // Modifier to check if policy is terminated
    modifier isNotTerminated() {
        if (s_policy.isTerminated) {
            revert PolicyTerminated();
        }
        _;
    }

    // constant decimals
    uint256 private constant DECIMALS = 10 ** 18;

    constructor(
        SharedData.Policy memory policy,
        address _link,
        address priceFeed
    ) ConfirmedOwner(msg.sender) {
        s_policy = policy;
        for (uint8 i = 0; i < s_policy.admins.length; i++) {
            s_admins.push(s_policy.admins[i]);
        }
        s_admins.push(s_policy.policyHolder.policyHolderWalletAddress);
        s_admins.push(address(this));
        s_priceFeed = AggregatorV3Interface(priceFeed);
        setChainlinkToken(_link);
        fee = (1 * LINK_DIVISIBILITY) / 10;
        s_timePassedSinceCreation = 0;
    }

    /****** BaseInsurancePolicy Functions ******/

    /**
    @dev function makeClaim
    @notice this function is used to make a claim on the policy, once the policy is claimed 
    now user can proceed to verify the details of claim by implementing methods like 
    DAO or Using API calls to validate data */

    function makeClaim() public onlyAdmin returns (bool claimed) {
        if (!s_policy.isTerminated) revert PolicyTerminated();
        if (s_policy.isPolicyActive) revert PolicyNotActive();

        if (s_policy.isClaimable) revert PolicyNotClaimable();

        // for single claimable policies
        if (!s_policy.hasClaimed) revert PolicyAlreadyClaimed();

        s_policy.hasClaimed = true;
        emit PolicyClaimed(address(this), block.timestamp, true);
        return true;
    }

    /**
    @dev function revivePolicy
    @notice this function is used to revive an inActive policy
    */
    function revivePolicy() public payable onlyAdmin {
        if (!s_policy.isTerminated) revert PolicyTerminated();
        if (!s_policy.isPolicyActive) revert PolicyActive();
        if (s_policy.hasClaimed) revert PolicyAlreadyClaimed();

        // if revivalPeriod is over then revert
        if (
            block.timestamp - s_lastPaymentTimestamp >
            s_policy.revivalRule.revivalPeriod + s_policy.gracePeriod //* seconds
        ) revert PolicyNotInGracePeriod();

        // amount should be close to revivalAmount
        if (s_policy.revivalRule.revivalAmount - msg.value < (1 * DECIMALS) / 10000)
            revert RevivalAmountNotCorrect();

        // set policy to active
        s_lastPaymentTimestamp = block.timestamp;
        s_policy.isPolicyActive = true;
        emit PolicyRevived(address(this), block.timestamp);
    }

    /**
    @dev function terminatePolicy
    @notice this function is used to terminate the policy and can only be called by the admins
    */
    function terminatePolicy() public onlyAdmin {
        if (s_policy.isTerminated) revert PolicyTerminated();
        s_policy.isTerminated = true;
        // sends the funds to the policyManager's wallet address
        s_policy.policyManagerAddress.transfer(address(this).balance);
        emit TerminatePolicy(address(this), block.timestamp);
    }

    /**
    @dev function withdraw
    @notice this function is used to withdraw the fund from policy, 
    function can be overidden to implement custom withdrawal logic 
    */
    function withdraw() public payable virtual onlyAdmin isNotTerminated {
        uint256 withdrawableAmount = s_policy.totalCoverageByPolicy;
        if (address(this).balance < withdrawableAmount)
            revert InsufficientBalance(address(this).balance, withdrawableAmount);
        s_policy.policyHolder.policyHolderWalletAddress.transfer(withdrawableAmount);
        setTermination(true);
        emit PolicyWithdraw(address(this), block.timestamp, withdrawableAmount);
    }

    /***** Chainlink Functionalities *****/
    /**
    @dev Chainlink Keepers Implementation
    @notice Chainlink Keepers is used to automatically check the policy activity. 
    It checks if the policy has been funded and takes decision based on the status of 
    the funding. The policy also gets automatically termainted when the timePassedSinceCreation
    surpasses the policyTenure
    
    @notice The function can be overidden to adjust according to the needs
    */

    function checkUpkeep(
        bytes calldata checkData
    ) external view override returns (bool upkeepNeeded, bytes memory performData) {
        // if policy is terminated then revert
        if (s_policy.isTerminated) {
            upkeepNeeded = false;
            performData = "Policy is terminated";
        }

        // if policy is not active then revert
        if (!s_policy.isPolicyActive) {
            // if revivalPeriod is over then revert
            if (
                block.timestamp - s_lastPaymentTimestamp >
                s_policy.gracePeriod + s_policy.revivalRule.revivalPeriod //* seconds
            ) {
                upkeepNeeded = false;
                performData = "Revival period is over";
            }
        }

        upkeepNeeded = (block.timestamp - s_lastPaymentTimestamp) > s_policy.timeInterval;
        performData = "Upkeep is needed";
    }

    function performUpkeep(bytes calldata performData) external override {
        s_policy.hasFundedForCurrentInterval = false;
        s_timePassedSinceCreation += s_policy.timeInterval;
        // if timePassedSinceCreation is greater than policyTenure then terminate policy as policy has matured
        if (s_timePassedSinceCreation >= s_policy.policyTenure) {
            terminatePolicy();
            emit PolicyMatured(address(this), block.timestamp);
        } else {
            // if gracePeriod is over then revert
            if (
                block.timestamp - s_lastPaymentTimestamp > s_policy.gracePeriod //* seconds
            ) s_policy.isPolicyActive = false;

            // if revivalPeriod is over then revert
            if (
                block.timestamp - s_lastPaymentTimestamp >
                s_policy.gracePeriod + s_policy.revivalRule.revivalPeriod //* seconds
            ) terminatePolicy();
        }
    }

    // fallback function
    receive() external payable {
        // if policy is terminated then revert
        if (!s_policy.isTerminated) revert PolicyTerminated();

        // if policy is not active then revert
        if (!s_policy.isPolicyActive) {
            // if revivalPeriod is over then revert
            if (
                block.timestamp - s_lastPaymentTimestamp >
                s_policy.revivalRule.revivalPeriod + s_policy.gracePeriod //* seconds
            ) revert PolicyNotInGracePeriod();

            // amount should be close to revivalAmount
            if (s_policy.revivalRule.revivalAmount - msg.value > (1 * DECIMALS) / 10000)
                revert RevivalAmountNotCorrect();

            // set policy to active
            s_lastPaymentTimestamp = block.timestamp;
            s_policy.isPolicyActive = true;
            return;
        }

        // if premiumToBePaid is close to msg.value then revert
        if (s_policy.premiumToBePaid - msg.value > (1 * DECIMALS) / 10000)
            revert PremiumAmountNotCorrect();

        // set lastTimestamp to current block.timestamp
        s_lastPaymentTimestamp = block.timestamp;
        s_policy.hasFundedForCurrentInterval = true;
    }

    /**
    @dev Chainlink Any API Implementation
    @param url the url of the API
    @param path the path to the field that you want to retrieve in the JSON body of the response
    @param jobId the jobId of the Chainlink node depending on the type of data you want to get
    @param oracle the associated oracle address to the API that you want to use
    @notice  The requestVolumeData function can be used to call any API and get the response
    
    @notice The function can be overidden to adjust according to the needs, 
    for example if you want to get multiple variables data*/
    function requestVolumeData(
        string memory url,
        string memory path,
        bytes32 jobId,
        address oracle
    ) public returns (bytes32 requestId) {
        setChainlinkOracle(oracle);
        Chainlink.Request memory req = buildChainlinkRequest(
            jobId,
            address(this),
            this.fulfill.selector
        );

        // Set the URL to perform the GET request on
        req.add("get", url);
        req.add("path", path);

        int256 timesAmount = 10 ** 18;
        req.addInt("times", timesAmount);

        // Sends the request
        return sendChainlinkRequest(req, fee);
    }

    /**
     * Receive the response in the form of uint256
     */
    function fulfill(
        bytes32 _requestId,
        uint256 /* _volume */
    ) public recordChainlinkFulfillment(_requestId) {
        // override this function to implement callback functionality
    }

    // Setter functions
    function setTermination(bool isTerminate) public onlyAdmin {
        s_policy.isTerminated = isTerminate;
    }

    function setClaimable(bool isClaimable) public onlyAdmin {
        s_policy.isClaimable = isClaimable;
    }

    function setClaimed(bool hasClaimed) public onlyAdmin {
        s_policy.hasClaimed = hasClaimed;
    }

    function setPolicyActive(bool isPolicyActive) public onlyAdmin {
        s_policy.isPolicyActive = isPolicyActive;
    }

    function sethasFundedForCurrentInterval(bool hasFundedForCurrentInterval) public onlyAdmin {
        s_policy.hasFundedForCurrentInterval = hasFundedForCurrentInterval;
    }

    // getter functions
    function getPolicyHolderDetails()
        public
        view
        returns (SharedData.HumanDetails memory policyHolderDetails)
    {
        return s_policy.policyHolder;
    }

    function getPolicyTenure() public view returns (uint128 policyTenure) {
        return s_policy.policyTenure;
    }

    function getGracePeriod() public view returns (uint128 gracePeriod) {
        return s_policy.gracePeriod;
    }

    function getTimeBeforeCommencement() public view returns (uint128 time) {
        return s_policy.timeBeforeCommencement;
    }

    function getPremiumToBePaid() public view returns (uint256 premiumToBePaid) {
        return s_policy.premiumToBePaid;
    }

    function getTotalCoverageByPolicy() public view returns (uint256 coverage) {
        return s_policy.totalCoverageByPolicy;
    }

    function getPolicyDetails() public view returns (string memory policyDetails) {
        return s_policy.policyDetails;
    }

    function getRevivalRule() public view returns (SharedData.RevivalRule memory revivalRule) {
        return s_policy.revivalRule;
    }

    function getHasClaimed() public view returns (bool hasClaimed) {
        return s_policy.hasClaimed;
    }

    function getIsPolicyActive() public view returns (bool isPolicyActive) {
        return s_policy.isPolicyActive;
    }

    function getIsClaimable() public view returns (bool isClaimable) {
        return s_policy.isClaimable;
    }

    function getIsTerminated() public view returns (bool isTerminated) {
        return s_policy.isTerminated;
    }

    function getPolicyHolderWalletAddress()
        public
        view
        returns (address payable policyHolderWalletAddress)
    {
        return s_policy.policyHolder.policyHolderWalletAddress;
    }

    function getPolicyType() public view returns (SharedData.PolicyType policyType) {
        return s_policy.policyType;
    }

    /**
    @dev Chainlink PriceFeed Implementation
    note This function gets the premium to be paid in USD using Chainlink PriceFeed
    */
    function getPremiuminUSD() public view returns (uint256 convertedAmount) {
        (, int256 answer, , , ) = s_priceFeed.latestRoundData();
        uint256 ethPriceInUsd = uint256(answer);
        // ETH/USD in 10^8 digit
        uint256 usdAmount = (s_policy.premiumToBePaid * ethPriceInUsd) / DECIMALS;
        return usdAmount;
    }

    function getAdmins() public view returns (address[] memory admins) {
        return s_admins;
    }
}

```