# Babylon & Jeju AI Governance: Technical Specification

**Version:** 2.0  
**Status:** Technical Implementation Guide

---

## 1. SMART CONTRACT ARCHITECTURE

### 1.1 Core Contracts

#### OpinionStaking.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "./IEIP8004Registry.sol"; // EIP-8004 interface

contract OpinionStaking is Initializable, UUPSUpgradeable, ReentrancyGuardUpgradeable, PausableUpgradeable {
    IEIP8004Registry public eip8004Registry;
    
    // Multi-token support
    mapping(address => bool) public acceptedTokens; // Token address => accepted
    mapping(address => uint256) public tokenWeights; // Token address => weight multiplier (basis points, 10000 = 1x)
    address[] public allAcceptedTokens;
    
    // Opinion scoping (network-level, app-level, cross-app)
    enum OpinionScope {
        Network,    // Network-wide (Jeju Network)
        App,        // App-specific (Gateway, Bazaar, Compute, etc.)
        CrossApp    // Affects multiple apps
    }
    
    struct Opinion {
        bytes32 textHash; // Hash of text (stored on IPFS)
        string ipfsHash; // IPFS CID for full text
        address creator;
        uint256 totalStaked; // Total staked (normalized across tokens)
        uint256 timestamp;
        uint256 clusterId; // For opinion clustering
        uint256 eip8004Reputation; // EIP-8004 reputation at creation time
        OpinionScope scope; // Network, App, or CrossApp
        address appAddress; // If app-specific, address of app contract (0x0 for network-level)
        address[] acceptedTokensForOpinion; // Tokens accepted for this opinion (empty = all accepted tokens)
        bool active;
        bool validated; // Validated based on EIP-8004 reputation
    }
    
    struct Stake {
        address staker;
        address token; // Token address used for staking
        uint256 amount; // Amount in token's native units
        uint256 normalizedAmount; // Amount normalized by token weight
        uint256 timestamp;
        uint256 unlockTime; // Cooldown period
    }
    
    mapping(uint256 => Opinion) public opinions;
    mapping(uint256 => mapping(address => Stake)) public stakes; // opinionId => staker => stake
    mapping(uint256 => mapping(address => uint256)) public stakesByToken; // opinionId => token => total staked
    mapping(address => uint256) public reputation; // EIP-8004 reputation score
    mapping(uint256 => uint256[]) public clusterMembers; // Opinion clusters
    
    uint256 public opinionCount;
    uint256 public constant COOLDOWN_PERIOD = 1 days; // Optimized for Jeju's 200ms blocks
    uint256 public constant MIN_STAKE_NORMALIZED = 1e18; // Minimum normalized stake
    
    event OpinionCreated(uint256 indexed opinionId, address creator, bytes32 textHash, string ipfsHash, uint256 reputation, bool validated, OpinionScope scope, address appAddress);
    event TokensStaked(uint256 indexed opinionId, address staker, address token, uint256 amount, uint256 normalizedAmount);
    event TokensUnstaked(uint256 indexed opinionId, address staker, address token, uint256 amount, uint256 normalizedAmount);
    event OpinionClustered(uint256 indexed clusterId, uint256[] opinionIds);
    event TokenAdded(address indexed token, uint256 weight);
    event TokenRemoved(address indexed token);
    
    function initialize(address _eip8004Registry) public initializer {
        __ReentrancyGuard_init();
        __Pausable_init();
        __UUPSUpgradeable_init();
        eip8004Registry = IEIP8004Registry(_eip8004Registry);
    }
    
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
    
    // Token management
    function addAcceptedToken(address _token, uint256 _weight) external onlyOwner {
        require(_token != address(0), "Invalid token");
        require(_weight > 0, "Weight must be > 0");
        
        if (!acceptedTokens[_token]) {
            acceptedTokens[_token] = true;
            allAcceptedTokens.push(_token);
        }
        tokenWeights[_token] = _weight; // Basis points (10000 = 1x)
        
        emit TokenAdded(_token, _weight);
    }
    
    function removeAcceptedToken(address _token) external onlyOwner {
        require(acceptedTokens[_token], "Token not accepted");
        
        // Note: Removing a token sets its weight to 0, but existing stakes can still be unstaked
        // The unstake function uses proportional calculation from stored normalizedAmount,
        // so accounting remains correct even after token removal
        acceptedTokens[_token] = false;
        tokenWeights[_token] = 0;
        
        emit TokenRemoved(_token);
    }
    
    function createOpinion(
        string memory _text,
        string memory _ipfsHash,
        OpinionScope _scope,
        address _appAddress,
        address[] memory _acceptedTokens
    ) external whenNotPaused returns (uint256) {
        require(bytes(_text).length >= 20 && bytes(_text).length <= 1000, "Invalid text length");
        require(_scope != OpinionScope.App || _appAddress != address(0), "App address required for app-scoped opinion");
        
        // Validate accepted tokens (if specified)
        if (_acceptedTokens.length > 0) {
            for (uint i = 0; i < _acceptedTokens.length; i++) {
                require(acceptedTokens[_acceptedTokens[i]], "Token not accepted");
            }
        }
        
        // Get EIP-8004 reputation
        uint256 reputation = eip8004Registry.getReputation(msg.sender);
        bool validated = reputation >= MIN_REPUTATION_FOR_VALIDATION;
        
        bytes32 textHash = keccak256(bytes(_text));
        uint256 opinionId = opinionCount++;
        
        opinions[opinionId] = Opinion({
            textHash: textHash,
            ipfsHash: _ipfsHash,
            creator: msg.sender,
            totalStaked: 0,
            timestamp: block.timestamp,
            clusterId: 0, // Will be set by AI clustering
            eip8004Reputation: reputation,
            scope: _scope,
            appAddress: _appAddress,
            acceptedTokensForOpinion: _acceptedTokens,
            active: true,
            validated: validated
        });
        
        emit OpinionCreated(opinionId, msg.sender, textHash, _ipfsHash, reputation, validated, _scope, _appAddress);
        return opinionId;
    }
    
    function validateOpinion(uint256 _opinionId) external {
        Opinion storage opinion = opinions[_opinionId];
        uint256 currentReputation = eip8004Registry.getReputation(opinion.creator);
        
        if (currentReputation >= MIN_REPUTATION_FOR_VALIDATION && !opinion.validated) {
            opinion.validated = true;
            opinion.eip8004Reputation = currentReputation;
            emit OpinionValidated(_opinionId, currentReputation);
        }
    }
    
    function stakeOnOpinion(uint256 _opinionId, address _token, uint256 _amount) external nonReentrant {
        Opinion storage opinion = opinions[_opinionId];
        require(opinion.active, "Opinion not active");
        
        // Check if token is accepted for this opinion
        if (opinion.acceptedTokensForOpinion.length > 0) {
            bool tokenAccepted = false;
            for (uint i = 0; i < opinion.acceptedTokensForOpinion.length; i++) {
                if (opinion.acceptedTokensForOpinion[i] == _token) {
                    tokenAccepted = true;
                    break;
                }
            }
            require(tokenAccepted, "Token not accepted for this opinion");
        } else {
            require(acceptedTokens[_token], "Token not accepted");
        }
        
        // Calculate normalized amount
        uint256 tokenWeight = tokenWeights[_token];
        require(tokenWeight > 0, "Token weight not set");
        uint256 normalizedAmount = (_amount * tokenWeight) / 10000;
        require(normalizedAmount >= MIN_STAKE_NORMALIZED, "Stake too small");
        
        Stake storage stake = stakes[_opinionId][msg.sender];
        require(stake.unlockTime == 0 || block.timestamp >= stake.unlockTime, "Cooldown active");
        
        IERC20(_token).transferFrom(msg.sender, address(this), _amount);
        
        // Update stake (if existing stake is in different token, replace it)
        bool isTokenSwitch = stake.amount > 0 && stake.token != _token;
        if (isTokenSwitch) {
            // Unstake previous token first
            IERC20(stake.token).transfer(msg.sender, stake.amount);
            opinions[_opinionId].totalStaked -= stake.normalizedAmount;
            stakesByToken[_opinionId][stake.token] -= stake.amount;
        }
        
        stake.token = _token;
        // If switching tokens, assign; if same token, add to existing
        if (isTokenSwitch) {
            stake.amount = _amount;
            stake.normalizedAmount = normalizedAmount;
        } else {
            stake.amount += _amount;
            stake.normalizedAmount += normalizedAmount;
        }
        stake.timestamp = block.timestamp;
        stake.unlockTime = block.timestamp + COOLDOWN_PERIOD;
        
        opinions[_opinionId].totalStaked += normalizedAmount;
        stakesByToken[_opinionId][_token] += _amount;
        
        emit TokensStaked(_opinionId, msg.sender, _token, _amount, normalizedAmount);
    }
    
    function unstakeFromOpinion(uint256 _opinionId, uint256 _amount) external nonReentrant {
        Stake storage stake = stakes[_opinionId][msg.sender];
        require(stake.amount >= _amount, "Insufficient stake");
        require(block.timestamp >= stake.unlockTime, "Cooldown active");
        
        // Calculate normalized amount proportionally from stored normalizedAmount
        // This ensures correct accounting even if token weight was changed or removed
        // Formula: (unstakeAmount / totalStakeAmount) * storedNormalizedAmount
        uint256 normalizedAmount;
        if (stake.amount > 0) {
            normalizedAmount = (_amount * stake.normalizedAmount) / stake.amount;
        } else {
            // Fallback: if stake.amount is 0 but normalizedAmount exists, use current weight
            // This should not happen in normal operation, but provides safety
            uint256 tokenWeight = tokenWeights[stake.token];
            normalizedAmount = (_amount * tokenWeight) / 10000;
        }
        
        stake.amount -= _amount;
        stake.normalizedAmount -= normalizedAmount;
        opinions[_opinionId].totalStaked -= normalizedAmount;
        stakesByToken[_opinionId][stake.token] -= _amount;
        
        IERC20(stake.token).transfer(msg.sender, _amount);
        
        emit TokensUnstaked(_opinionId, msg.sender, stake.token, _amount, normalizedAmount);
    }
    
    function getOpinionWeight(uint256 _opinionId) external view returns (uint256) {
        Opinion memory opinion = opinions[_opinionId];
        uint256 baseWeight = opinion.totalStaked;
        
        // Time decay: recent stakes weighted higher
        uint256 timeDecay = calculateTimeDecay(opinion.timestamp);
        
        // EIP-8004 reputation weighting
        uint256 repMultiplier = 100; // Base multiplier
        if (opinion.validated) {
            // Validated opinions get reputation boost (1-2x multiplier)
            repMultiplier = 100 + (opinion.eip8004Reputation / 10);
        } else {
            // Unvalidated opinions get reduced weight
            repMultiplier = 50; // 0.5x multiplier
        }
        
        return (baseWeight * timeDecay * repMultiplier) / (100 * 100);
    }
    
    function calculateTimeDecay(uint256 _timestamp) internal view returns (uint256) {
        uint256 age = block.timestamp - _timestamp;
        if (age < 7 days) return 100; // Full weight
        if (age < 30 days) return 80;  // 80% weight
        if (age < 90 days) return 60;  // 60% weight
        return 40; // 40% weight after 90 days
    }
    
    uint256 public constant MIN_REPUTATION_FOR_VALIDATION = 50; // EIP-8004 reputation threshold
    
    event OpinionCreated(uint256 indexed opinionId, address creator, bytes32 textHash, string ipfsHash, uint256 reputation, bool validated);
    event OpinionValidated(uint256 indexed opinionId, uint256 reputation);
    
    function clusterOpinions(uint256[] memory _opinionIds, uint256 _clusterId) external {
        // Only callable by AI CEO or authorized agent
        clusterMembers[_clusterId] = _opinionIds;
        for (uint i = 0; i < _opinionIds.length; i++) {
            opinions[_opinionIds[i]].clusterId = _clusterId;
        }
        emit OpinionClustered(_clusterId, _opinionIds);
    }
}
```

#### ProposalExecution.sol (Upgradeable with Emergency Shutdown)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/ReentrancyGuardUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/security/PausableUpgradeable.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";

contract ProposalExecution is Initializable, UUPSUpgradeable, OwnableUpgradeable, ReentrancyGuardUpgradeable, PausableUpgradeable {
    struct Proposal {
        string description;
        address proposer; // Agent (EIP-8004) or human address
        bytes executionData;
        uint256 executionTime;
        uint256 vetoDeadline;
        bool executed;
        bool vetoed;
        uint256 totalVetoVotes;
    }
    
    struct VetoVote {
        address voter;
        uint256 amount;
        bool voted;
    }
    
    mapping(uint256 => Proposal) public proposals;
    mapping(uint256 => mapping(address => VetoVote)) public vetoVotes;
    mapping(address => bool) public authorizedAgents; // EIP-8004 agent addresses
    
    uint256 public proposalCount;
    uint256 public constant VETO_THRESHOLD_PERCENT = 10; // 10% of total supply
    uint256 public constant VETO_WINDOW = 7 days;
    uint256 public constant EMERGENCY_SHUTDOWN_THRESHOLD = 30; // 30% of total supply
    uint256 public constant EMERGENCY_SHUTDOWN_RECOVERY_THRESHOLD = 50; // 50% to resume
    
    IERC20 public governanceToken;
    OpinionStaking public opinionStaking;
    
    bool public emergencyShutdown;
    uint256 public emergencyShutdownStake;
    mapping(address => uint256) public emergencyShutdownVotes;
    
    function initialize(address _governanceToken, address _opinionStaking) public initializer {
        __Ownable_init();
        __ReentrancyGuard_init();
        __Pausable_init();
        __UUPSUpgradeable_init();
        governanceToken = IERC20(_governanceToken);
        opinionStaking = OpinionStaking(_opinionStaking);
    }
    
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
    
    event ProposalCreated(uint256 indexed proposalId, address proposer, string description);
    event ProposalExecuted(uint256 indexed proposalId);
    event ProposalVetoed(uint256 indexed proposalId, uint256 vetoVotes);
    event VetoVoteCast(uint256 indexed proposalId, address voter, uint256 amount);
    
    function createProposal(
        string memory _description,
        bytes memory _executionData,
        uint256 _executionTime
    ) external returns (uint256) {
        require(
            authorizedAgents[msg.sender] || msg.sender == owner(),
            "Not authorized"
        );
        
        uint256 proposalId = proposalCount++;
        proposals[proposalId] = Proposal({
            description: _description,
            proposer: msg.sender,
            executionData: _executionData,
            executionTime: _executionTime,
            vetoDeadline: _executionTime + VETO_WINDOW,
            executed: false,
            vetoed: false,
            totalVetoVotes: 0
        });
        
        emit ProposalCreated(proposalId, msg.sender, _description);
        return proposalId;
    }
    
    struct FinancialTransaction {
        address recipient;
        uint256 amount;
        bool isKnownAddress;
        bool isStandardPayment;
        uint256 timestamp;
    }
    
    mapping(address => bool) public knownAddresses; // Known addresses for standard payments
    mapping(address => bool) public trustedContributors; // Trusted contributors can speed up/pause
    uint256 public constant STANDARD_PAYMENT_THRESHOLD = 10000e18; // 10K tokens
    uint256 public constant BIG_PAYMENT_THRESHOLD = 100000e18; // 100K tokens
    
    function executeProposal(uint256 _proposalId) external nonReentrant whenNotPaused {
        require(!emergencyShutdown, "Emergency shutdown active");
        
        Proposal storage proposal = proposals[_proposalId];
        require(!proposal.executed, "Already executed");
        require(!proposal.vetoed, "Proposal vetoed");
        require(block.timestamp >= proposal.executionTime, "Too early");
        require(block.timestamp <= proposal.vetoDeadline, "Veto window closed");
        
        // Check if veto threshold met
        uint256 totalSupply = governanceToken.totalSupply();
        uint256 vetoThreshold = (totalSupply * VETO_THRESHOLD_PERCENT) / 100;
        
        if (proposal.totalVetoVotes >= vetoThreshold) {
            proposal.vetoed = true;
            emit ProposalVetoed(_proposalId, proposal.totalVetoVotes);
            return;
        }
        
        // Check if this is a financial transaction
        FinancialTransaction memory tx = this.parseFinancialTransaction(proposal.executionData);
        
        if (tx.amount > 0) {
            // Financial transaction logic
            if (this.isStandardPayment(tx)) {
                // Standard payment: execute immediately, no human review
                // Known address, regular payment, below threshold
            } else {
                // Unusual/big/new payment: require 7-day veto period
                require(block.timestamp >= proposal.executionTime + VETO_WINDOW, "Veto period not passed");
            }
        }
        
        // Simulate execution first (off-chain, but verified)
        // In production, this would call a simulation service
        
        // Execute proposal
        (bool success, ) = address(this).call(proposal.executionData);
        require(success, "Execution failed");
        
        proposal.executed = true;
        emit ProposalExecuted(_proposalId);
    }
    
    function isStandardPayment(FinancialTransaction memory _tx) internal view returns (bool) {
        return _tx.isKnownAddress && 
               _tx.isStandardPayment && 
               _tx.amount <= STANDARD_PAYMENT_THRESHOLD;
    }
    
    function speedUpExecution(uint256 _proposalId) external {
        require(trustedContributors[msg.sender], "Not a trusted contributor");
        Proposal storage proposal = proposals[_proposalId];
        // Trusted contributor can speed up execution
        proposal.executionTime = block.timestamp;
    }
    
    function pauseExecution(uint256 _proposalId) external {
        require(trustedContributors[msg.sender], "Not a trusted contributor");
        Proposal storage proposal = proposals[_proposalId];
        // Trusted contributor can pause execution
        proposal.executionTime = type(uint256).max; // Effectively pause
    }
    
    function addKnownAddress(address _address) external onlyOwner {
        knownAddresses[_address] = true;
        emit KnownAddressAdded(_address);
    }
    
    function removeKnownAddress(address _address) external onlyOwner {
        knownAddresses[_address] = false;
        emit KnownAddressRemoved(_address);
    }
    
    function addTrustedContributor(address _contributor) external onlyOwner {
        trustedContributors[_contributor] = true;
        emit TrustedContributorAdded(_contributor);
    }
    
    function removeTrustedContributor(address _contributor) external onlyOwner {
        trustedContributors[_contributor] = false;
        emit TrustedContributorRemoved(_contributor);
    }
    
    function parseFinancialTransaction(bytes memory _executionData) external pure returns (FinancialTransaction memory) {
        // Parse execution data to extract financial transaction details
        // This is a simplified version - in production, would need proper ABI decoding
        // Assumes executionData contains: recipient (address), amount (uint256)
        
        require(_executionData.length >= 64, "Invalid execution data");
        
        address recipient;
        uint256 amount;
        
        assembly {
            recipient := mload(add(_executionData, 20))
            amount := mload(add(_executionData, 52))
        }
        
        return FinancialTransaction({
            recipient: recipient,
            amount: amount,
            isKnownAddress: false, // Will be set by caller
            isStandardPayment: false, // Will be determined by caller
            timestamp: block.timestamp
        });
    }
    
    event KnownAddressAdded(address indexed address);
    event KnownAddressRemoved(address indexed address);
    event TrustedContributorAdded(address indexed contributor);
    event TrustedContributorRemoved(address indexed contributor);
}
    
    function triggerEmergencyShutdown(uint256 _amount) external nonReentrant {
        require(!emergencyShutdown, "Already shutdown");
        
        governanceToken.transferFrom(msg.sender, address(this), _amount);
        emergencyShutdownVotes[msg.sender] += _amount;
        emergencyShutdownStake += _amount;
        
        uint256 totalSupply = governanceToken.totalSupply();
        uint256 threshold = (totalSupply * EMERGENCY_SHUTDOWN_THRESHOLD) / 100;
        
        if (emergencyShutdownStake >= threshold) {
            emergencyShutdown = true;
            _pause(); // Pause all operations
            emit EmergencyShutdownTriggered(emergencyShutdownStake);
        }
        
        emit EmergencyShutdownVote(msg.sender, _amount, emergencyShutdownStake);
    }
    
    function resumeAfterShutdown() external onlyOwner {
        require(emergencyShutdown, "Not in shutdown");
        
        // Require governance vote to resume (50% threshold)
        uint256 totalSupply = governanceToken.totalSupply();
        uint256 recoveryThreshold = (totalSupply * EMERGENCY_SHUTDOWN_RECOVERY_THRESHOLD) / 100;
        
        // In production, this would check governance vote
        // For now, owner can resume (should be replaced with governance)
        
        emergencyShutdown = false;
        _unpause();
        emit EmergencyShutdownResumed();
    }
    
    event EmergencyShutdownTriggered(uint256 totalStake);
    event EmergencyShutdownVote(address indexed voter, uint256 amount, uint256 totalStake);
    event EmergencyShutdownResumed();
    
    function vetoProposal(uint256 _proposalId, uint256 _amount) external {
        Proposal storage proposal = proposals[_proposalId];
        require(!proposal.executed, "Already executed");
        require(block.timestamp <= proposal.vetoDeadline, "Veto window closed");
        
        VetoVote storage vote = vetoVotes[_proposalId][msg.sender];
        require(!vote.voted, "Already voted");
        
        governanceToken.transferFrom(msg.sender, address(this), _amount);
        
        vote.voter = msg.sender;
        vote.amount = _amount;
        vote.voted = true;
        proposal.totalVetoVotes += _amount;
        
        emit VetoVoteCast(_proposalId, msg.sender, _amount);
        
        // Check if threshold met
        uint256 totalSupply = governanceToken.totalSupply();
        uint256 vetoThreshold = (totalSupply * VETO_THRESHOLD_PERCENT) / 100;
        
        if (proposal.totalVetoVotes >= vetoThreshold) {
            proposal.vetoed = true;
            emit ProposalVetoed(_proposalId, proposal.totalVetoVotes);
        }
    }
    
    function authorizeAgent(address _agent) external onlyOwner {
        authorizedAgents[_agent] = true;
    }
    
    function revokeAgent(address _agent) external onlyOwner {
        authorizedAgents[_agent] = false;
    }
}
```

#### WorkTracking.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/access/Ownable.sol";

contract WorkTracking is Ownable {
    struct Work {
        uint256 workId;
        uint8 workType; // 0 = PR, 1 = Proposal, 2 = Implementation
        address agent; // EIP-8004 agent address
        string title;
        string repository; // For PRs
        uint256 prNumber; // For PRs
        uint256 createdAt;
        uint256 mergedAt;
        bool merged;
        uint256[] relatedOpinions; // Opinion IDs this work addresses
        uint256 credit; // Credit assigned (basis points)
        uint256 qualityScore; // Quality score for competition
    }
    
    mapping(uint256 => Work) public works;
    mapping(address => uint256[]) public agentWorks; // Agent address => work IDs
    mapping(uint256 => uint256[]) public opinionWorks; // Opinion ID => work IDs
    mapping(uint256 => uint256[]) public competingWorks; // Task ID => competing work IDs
    uint256 public workCount;
    
    event WorkRegistered(uint256 indexed workId, address indexed agent, uint8 workType, string title);
    event WorkMerged(uint256 indexed workId, address indexed agent);
    event WorkCreditAssigned(uint256 indexed workId, address indexed agent, uint256 credit);
    event CompetingWorkRegistered(uint256 indexed taskId, uint256 indexed workId);
    
    function registerWork(
        uint8 _workType,
        address _agent,
        string memory _title,
        string memory _repository,
        uint256 _prNumber,
        uint256[] memory _relatedOpinions,
        uint256 _taskId
    ) external onlyOwner returns (uint256) {
        require(_agent != address(0), "Invalid agent");
        
        uint256 workId = workCount++;
        works[workId] = Work({
            workId: workId,
            workType: _workType,
            agent: _agent,
            title: _title,
            repository: _repository,
            prNumber: _prNumber,
            createdAt: block.timestamp,
            mergedAt: 0,
            merged: false,
            relatedOpinions: _relatedOpinions,
            credit: 0,
            qualityScore: 0
        });
        
        agentWorks[_agent].push(workId);
        for (uint i = 0; i < _relatedOpinions.length; i++) {
            opinionWorks[_relatedOpinions[i]].push(workId);
        }
        
        // Track competing works
        if (_taskId > 0) {
            competingWorks[_taskId].push(workId);
            emit CompetingWorkRegistered(_taskId, workId);
        }
        
        emit WorkRegistered(workId, _agent, _workType, _title);
        return workId;
    }
    
    function markWorkMerged(uint256 _workId) external onlyOwner {
        Work storage work = works[_workId];
        require(!work.merged, "Work already merged");
        
        work.merged = true;
        work.mergedAt = block.timestamp;
        
        emit WorkMerged(_workId, work.agent);
    }
    
    function assignWorkCredit(uint256 _workId, uint256 _credit) external onlyOwner {
        Work storage work = works[_workId];
        require(work.merged, "Work not merged");
        
        work.credit = _credit;
        
        emit WorkCreditAssigned(_workId, work.agent, _credit);
    }
    
    function setQualityScore(uint256 _workId, uint256 _score) external onlyOwner {
        works[_workId].qualityScore = _score;
    }
    
    function getAgentWorks(address _agent) external view returns (uint256[] memory) {
        return agentWorks[_agent];
    }
    
    function getCompetingWorks(uint256 _taskId) external view returns (uint256[] memory) {
        return competingWorks[_taskId];
    }
    
    function getMergedWorks(address _agent) external view returns (uint256[] memory) {
        uint256[] memory allWorks = agentWorks[_agent];
        uint256[] memory merged = new uint256[](allWorks.length);
        uint256 count = 0;
        
        for (uint i = 0; i < allWorks.length; i++) {
            if (works[allWorks[i]].merged) {
                merged[count] = allWorks[i];
                count++;
            }
        }
        
        uint256[] memory result = new uint256[](count);
        for (uint i = 0; i < count; i++) {
            result[i] = merged[i];
        }
        
        return result;
    }
}
```

#### DeepFunding.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/access/Ownable.sol";
import "./WorkTracking.sol";

contract DeepFunding is Ownable {
    IERC20 public immutable governanceToken;
    IERC20 public immutable treasuryToken;
    WorkTracking public workTracking;
    
    struct Contribution {
        address contributor;
        string description;
        uint256 credit; // Credit percentage (basis points, 10000 = 100%)
        address[] dependencies; // Upstream dependencies
        ContributionType contributionType;
    }
    
    enum ContributionType {
        Code,           // Code commits, PRs
        Proposal,       // Governance proposals
        Documentation,  // Documentation
        Support,        // Community support
        Research,       // Research contributions
        Other
    }
    
    struct DependencyGraph {
        address node;
        uint256 weight; // Edge weight (basis points)
        address[] children;
    }
    
    mapping(address => Contribution) public contributions;
    mapping(address => DependencyGraph) public dependencyGraph;
    mapping(address => uint256) public totalDistributed;
    mapping(address => uint256) public agentTotalCredit; // Agent address => total credit
    
    event ContributionRegistered(address indexed contributor, string description, uint256 credit, ContributionType contributionType);
    event DependencyAdded(address indexed parent, address indexed dependency, uint256 weight);
    event RewardsDistributed(address indexed contributor, uint256 amount);
    event PRCreditAssigned(address indexed agent, uint256 prNumber, uint256 credit);
    
    constructor(address _governanceToken, address _treasuryToken, address _workTracking) {
        governanceToken = IERC20(_governanceToken);
        treasuryToken = IERC20(_treasuryToken);
        workTracking = WorkTracking(_workTracking);
    }
    
    function registerContribution(
        address _contributor,
        string memory _description,
        uint256 _credit,
        address[] memory _dependencies,
        ContributionType _contributionType
    ) external onlyOwner {
        contributions[_contributor] = Contribution({
            contributor: _contributor,
            description: _description,
            credit: _credit,
            dependencies: _dependencies,
            contributionType: _contributionType
        });
        
        emit ContributionRegistered(_contributor, _description, _credit, _contributionType);
    }
    
    function registerWorkContribution(
        address _agent,
        uint256 _workId,
        uint256 _credit
    ) external onlyOwner {
        // Register work as contribution
        Work memory work = workTracking.works(_workId);
        string memory workDesc = work.workType == 0 
            ? string(abi.encodePacked("PR #", work.prNumber))
            : (work.workType == 1 ? "Proposal" : "Implementation");
        
        address[] memory emptyDeps = new address[](0);
        contributions[_agent] = Contribution({
            contributor: _agent,
            description: workDesc,
            credit: _credit,
            dependencies: emptyDeps,
            contributionType: ContributionType.Code
        });
        
        agentTotalCredit[_agent] += _credit;
        
        emit PRCreditAssigned(_agent, _workId, _credit);
        emit ContributionRegistered(_agent, workDesc, _credit, ContributionType.Code);
    }
    
    function addDependency(
        address _parent,
        address _dependency,
        uint256 _weight
    ) external onlyOwner {
        DependencyGraph storage graph = dependencyGraph[_parent];
        if (graph.node == address(0)) {
            graph.node = _parent;
        }
        graph.children.push(_dependency);
        graph.weight = _weight;
        
        emit DependencyAdded(_parent, _dependency, _weight);
    }
    
    function distributeRewards(
        address _rootNode,
        uint256 _totalAmount
    ) external onlyOwner {
        _distributeRecursive(_rootNode, _totalAmount, 10000); // Start with 100% (10000 basis points)
    }
    
    function _distributeRecursive(
        address _node,
        uint256 _totalAmount,
        uint256 _parentCredit
    ) internal {
        Contribution memory contrib = contributions[_node];
        if (contrib.contributor == address(0)) return;
        
        // Calculate this node's share
        uint256 nodeCredit = (_parentCredit * contrib.credit) / 10000;
        uint256 nodeAmount = (_totalAmount * nodeCredit) / 10000;
        
        // Distribute to contributor
        if (nodeAmount > 0 && contrib.contributor != address(0)) {
            treasuryToken.transfer(contrib.contributor, nodeAmount);
            totalDistributed[contrib.contributor] += nodeAmount;
            emit RewardsDistributed(contrib.contributor, nodeAmount);
        }
        
        // Recursively distribute to dependencies
        DependencyGraph memory graph = dependencyGraph[_node];
        uint256 remainingCredit = _parentCredit - nodeCredit;
        
        for (uint i = 0; i < graph.children.length; i++) {
            address dependency = graph.children[i];
            uint256 dependencyWeight = graph.weight / graph.children.length;
            uint256 dependencyCredit = (remainingCredit * dependencyWeight) / 10000;
            
            _distributeRecursive(dependency, _totalAmount, dependencyCredit);
        }
    }
    
    function getTotalCredit(address _node) external view returns (uint256) {
        return contributions[_node].credit;
    }
    
    function getDependencies(address _node) external view returns (address[] memory) {
        return dependencyGraph[_node].children;
    }
}
```

### 1.2 Integration Contracts

#### AICEOInterface.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

interface IAICEO {
    struct OpinionSummary {
        uint256 opinionId;
        string text;
        uint256 weight;
        uint256 clusterId;
    }
    
    struct Proposal {
        string description;
        bytes executionData;
        uint256 priority;
        address specializedAgent;
    }
    
    struct Decision {
        uint256 decisionId;
        string decision;
        string reasoning;
        uint256 timestamp;
        uint256[] relatedOpinions;
        uint256[] relatedProposals;
        uint256 goodVotes; // Tokens staked on "good decision"
        uint256 badVotes; // Tokens staked on "bad decision"
    }
    
    function synthesizeOpinions(
        OpinionSummary[] memory opinions
    ) external returns (Proposal[] memory);
    
    function makeDecision(
        uint256 proposalId,
        OpinionSummary[] memory relevantOpinions
    ) external returns (uint256 decisionId, bool execute, string memory reasoning);
    
    function publishDecision(Decision memory decision) external;
}
```

#### AICEDecisionFeed.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

contract AICEDecisionFeed {
    struct DecisionEntry {
        uint256 decisionId;
        string decision;
        string reasoning;
        uint256 timestamp;
        uint256[] relatedOpinions;
        uint256[] relatedProposals;
        uint256 goodVotes;
        uint256 badVotes;
        mapping(address => uint256) goodStakes; // Address => amount staked on "good"
        mapping(address => uint256) badStakes; // Address => amount staked on "bad"
    }
    
    mapping(uint256 => DecisionEntry) public decisions;
    uint256 public decisionCount;
    IERC20 public governanceToken;
    
    event DecisionPublished(uint256 indexed decisionId, string decision, string reasoning);
    event DecisionVote(uint256 indexed decisionId, address voter, bool isGood, uint256 amount);
    
    function publishDecision(
        string memory _decision,
        string memory _reasoning,
        uint256[] memory _relatedOpinions,
        uint256[] memory _relatedProposals
    ) external returns (uint256) {
        // Only callable by AI CEO
        uint256 decisionId = decisionCount++;
        DecisionEntry storage entry = decisions[decisionId];
        entry.decisionId = decisionId;
        entry.decision = _decision;
        entry.reasoning = _reasoning;
        entry.timestamp = block.timestamp;
        entry.relatedOpinions = _relatedOpinions;
        entry.relatedProposals = _relatedProposals;
        
        emit DecisionPublished(decisionId, _decision, _reasoning);
        return decisionId;
    }
    
    function voteOnDecision(uint256 _decisionId, bool _isGood, uint256 _amount) external {
        DecisionEntry storage entry = decisions[_decisionId];
        require(entry.decisionId != 0, "Decision not found");
        
        governanceToken.transferFrom(msg.sender, address(this), _amount);
        
        if (_isGood) {
            entry.goodStakes[msg.sender] += _amount;
            entry.goodVotes += _amount;
        } else {
            entry.badStakes[msg.sender] += _amount;
            entry.badVotes += _amount;
        }
        
        emit DecisionVote(_decisionId, msg.sender, _isGood, _amount);
    }
    
    function getDecisionFeed(uint256 _limit, uint256 _offset) external view returns (DecisionEntry[] memory) {
        // Returns recent decisions for feed display
        uint256 count = _limit > decisionCount ? decisionCount : _limit;
        DecisionEntry[] memory feed = new DecisionEntry[](count);
        
        for (uint i = 0; i < count; i++) {
            feed[i] = decisions[decisionCount - 1 - i - _offset];
        }
        
        return feed;
    }
}
```

#### AgentRegistry.sol (Connected to EIP-8004 on Jeju)
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "./IEIP8004Registry.sol"; // EIP-8004 interface (already on Jeju)

contract AgentRegistry {
    IEIP8004Registry public eip8004Registry;
    
    struct Agent {
        address agentAddress; // EIP-8004 identity
        string specialization; // Free-form specialization (not enum)
        string description;
        uint256 eip8004Reputation; // From EIP-8004 registry
        uint256 prsMerged;
        uint256 proposalsAccepted;
        uint256 totalCredit; // Deep funding credit
        bool active;
        bool verified; // Verified by community or admin
    }
    
    mapping(address => Agent) public agents;
    mapping(string => address[]) public agentsBySpecialization; // Dynamic specializations
    address[] public allAgents;
    
    event AgentRegistered(address indexed agent, string specialization, string description);
    event AgentPerformanceUpdated(address indexed agent, uint256 prsMerged, uint256 proposalsAccepted);
    event AgentVerified(address indexed agent, bool verified);
    
    constructor(address _eip8004Registry) {
        eip8004Registry = IEIP8004Registry(_eip8004Registry);
    }
    
    function registerAgent(
        address _agentAddress,
        string memory _specialization,
        string memory _description
    ) external {
        require(
            eip8004Registry.isAgent(_agentAddress),
            "Not a valid EIP-8004 agent"
        );
        
        uint256 reputation = eip8004Registry.getReputation(_agentAddress);
        
        agents[_agentAddress] = Agent({
            agentAddress: _agentAddress,
            specialization: _specialization,
            description: _description,
            eip8004Reputation: reputation,
            prsMerged: 0,
            proposalsAccepted: 0,
            totalCredit: 0,
            active: true,
            verified: reputation >= 50 // Auto-verify if reputation >= 50
        });
        
        agentsBySpecialization[_specialization].push(_agentAddress);
        allAgents.push(_agentAddress);
        
        emit AgentRegistered(_agentAddress, _specialization, _description);
    }
    
    function updatePerformance(
        address _agent,
        uint256 _prsMerged,
        uint256 _proposalsAccepted
    ) external {
        require(agents[_agent].active, "Agent not active");
        agents[_agent].prsMerged += _prsMerged;
        agents[_agent].proposalsAccepted += _proposalsAccepted;
        
        // Update EIP-8004 reputation from registry
        agents[_agent].eip8004Reputation = eip8004Registry.getReputation(_agent);
        
        emit AgentPerformanceUpdated(_agent, agents[_agent].prsMerged, agents[_agent].proposalsAccepted);
    }
    
    function getAgentsBySpecialization(string memory _specialization) external view returns (address[] memory) {
        return agentsBySpecialization[_specialization];
    }
    
    function searchAgents(string memory _query) external view returns (address[] memory) {
        // Search agents by specialization or description
        address[] memory results = new address[](allAgents.length);
        uint256 count = 0;
        
        for (uint i = 0; i < allAgents.length; i++) {
            Agent memory agent = agents[allAgents[i]];
            if (
                keccak256(bytes(agent.specialization)) == keccak256(bytes(_query)) ||
                this.containsString(agent.description, _query)
            ) {
                results[count] = allAgents[i];
                count++;
            }
        }
        
        // Resize array
        address[] memory finalResults = new address[](count);
        for (uint i = 0; i < count; i++) {
            finalResults[i] = results[i];
        }
        
        return finalResults;
    }
}
```

#### TrustedRegistry.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract TrustedRegistry is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    enum TrustType {
        Developer,      // Can merge PRs on GitHub
        Contributor     // Can speed up/pause financial transactions
    }
    
    struct TrustedEntity {
        address entity;
        TrustType trustType;
        string description;
        uint256 addedAt;
        bool active;
    }
    
    mapping(address => TrustedEntity) public trustedEntities;
    mapping(TrustType => address[]) public entitiesByType;
    address[] public allEntities;
    
    event TrustedEntityAdded(address indexed entity, TrustType trustType, string description);
    event TrustedEntityRemoved(address indexed entity);
    event TrustedEntityUpdated(address indexed entity, bool active);
    
    function initialize() public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
    }
    
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
    
    function addTrustedEntity(
        address _entity,
        TrustType _trustType,
        string memory _description
    ) external onlyOwner {
        require(trustedEntities[_entity].entity == address(0), "Already exists");
        
        trustedEntities[_entity] = TrustedEntity({
            entity: _entity,
            trustType: _trustType,
            description: _description,
            addedAt: block.timestamp,
            active: true
        });
        
        entitiesByType[_trustType].push(_entity);
        allEntities.push(_entity);
        
        emit TrustedEntityAdded(_entity, _trustType, _description);
    }
    
    function removeTrustedEntity(address _entity) external onlyOwner {
        require(trustedEntities[_entity].entity != address(0), "Not found");
        trustedEntities[_entity].active = false;
        emit TrustedEntityRemoved(_entity);
    }
    
    function updateTrustedEntity(address _entity, bool _active) external onlyOwner {
        require(trustedEntities[_entity].entity != address(0), "Not found");
        trustedEntities[_entity].active = _active;
        emit TrustedEntityUpdated(_entity, _active);
    }
    
    function isTrusted(address _entity, TrustType _trustType) external view returns (bool) {
        TrustedEntity memory entity = trustedEntities[_entity];
        return entity.active && 
               entity.entity != address(0) && 
               entity.trustType == _trustType;
    }
    
    function getTrustedDevelopers() external view returns (address[] memory) {
        return entitiesByType[TrustType.Developer];
    }
    
    function getTrustedContributors() external view returns (address[] memory) {
        return entitiesByType[TrustType.Contributor];
    }
}
```

#### KnownAddressRegistry.sol
```solidity
// SPDX-License-Identifier: MIT
pragma solidity ^0.8.20;

import "@openzeppelin/contracts-upgradeable/proxy/utils/Initializable.sol";
import "@openzeppelin/contracts-upgradeable/proxy/utils/UUPSUpgradeable.sol";
import "@openzeppelin/contracts-upgradeable/access/OwnableUpgradeable.sol";

contract KnownAddressRegistry is Initializable, UUPSUpgradeable, OwnableUpgradeable {
    struct KnownAddress {
        address addr;
        string description;
        uint256 addedAt;
        uint256 lastPayment;
        uint256 totalPayments;
        uint256 totalAmount;
        bool active;
    }
    
    mapping(address => KnownAddress) public knownAddresses;
    address[] public allAddresses;
    
    uint256 public constant MIN_PAYMENT_COUNT = 3; // Minimum payments to be considered "regular"
    uint256 public constant REGULAR_PAYMENT_WINDOW = 30 days; // Payments within this window are "regular"
    
    event KnownAddressAdded(address indexed addr, string description);
    event KnownAddressRemoved(address indexed addr);
    event PaymentRecorded(address indexed addr, uint256 amount);
    
    function initialize() public initializer {
        __Ownable_init();
        __UUPSUpgradeable_init();
    }
    
    function _authorizeUpgrade(address newImplementation) internal override onlyOwner {}
    
    function addKnownAddress(
        address _addr,
        string memory _description
    ) external onlyOwner {
        require(knownAddresses[_addr].addr == address(0), "Already exists");
        
        knownAddresses[_addr] = KnownAddress({
            addr: _addr,
            description: _description,
            addedAt: block.timestamp,
            lastPayment: 0,
            totalPayments: 0,
            totalAmount: 0,
            active: true
        });
        
        allAddresses.push(_addr);
        emit KnownAddressAdded(_addr, _description);
    }
    
    function removeKnownAddress(address _addr) external onlyOwner {
        require(knownAddresses[_addr].addr != address(0), "Not found");
        knownAddresses[_addr].active = false;
        emit KnownAddressRemoved(_addr);
    }
    
    function recordPayment(address _addr, uint256 _amount) external {
        // Only callable by ProposalExecution contract
        require(knownAddresses[_addr].addr != address(0), "Not a known address");
        
        KnownAddress storage ka = knownAddresses[_addr];
        ka.lastPayment = block.timestamp;
        ka.totalPayments++;
        ka.totalAmount += _amount;
        
        emit PaymentRecorded(_addr, _amount);
    }
    
    function isKnownAddress(address _addr) external view returns (bool) {
        return knownAddresses[_addr].active && knownAddresses[_addr].addr != address(0);
    }
    
    function isStandardPayment(address _addr) external view returns (bool) {
        KnownAddress memory ka = knownAddresses[_addr];
        if (!ka.active) return false;
        
        // Standard payment if:
        // - Has made at least MIN_PAYMENT_COUNT payments
        // - Last payment was within REGULAR_PAYMENT_WINDOW
        // - Or manually verified as standard
        
        bool hasRegularHistory = ka.totalPayments >= MIN_PAYMENT_COUNT;
        bool recentPayment = (block.timestamp - ka.lastPayment) <= REGULAR_PAYMENT_WINDOW;
        
        return hasRegularHistory && recentPayment;
    }
    
    function getAllKnownAddresses() external view returns (address[] memory) {
        return allAddresses;
    }
}
```

---

## 2. OFF-CHAIN COMPONENTS

### 2.1 AI CEO Service (Open Source, Single Implementation)

**Architecture:**
- **Single Open-Source Implementation:** One AI CEO, all code publicly available
- **Transparent Decision Feed:** All decisions published in real-time (like Twitter feed)
- **Community Feedback:** Token holders can stake tokens on decisions (good/bad)
- **Self-Correcting:** System adjusts based on community feedback
- **TEE Integration:** Uses Council app's TEE Service for hardware-attested decisions
- **Research Integration:** Uses Council app's ResearchAgent for deep analysis

**Decision Feed:**
- Every decision published immediately with reasoning
- Token holders can react: stake tokens on "good" or "bad"
- Feed shows: decision → community reactions → outcomes
- Pattern detection: if community consistently stakes against certain patterns, system adjusts

**API Endpoints:**

```typescript
// GET /api/opinions
// Returns all staked opinions with weights
interface Opinion {
  id: number;
  text: string;
  weight: bigint;
  clusterId: number;
  timestamp: number;
}

// POST /api/synthesize
// Synthesize opinions into prioritized actions
interface SynthesisRequest {
  opinions: Opinion[];
  protocolState: ProtocolState;
  historicalData: HistoricalData;
}

interface SynthesisResponse {
  prioritizedActions: Action[];
  reasoning: string;
  proposals: Proposal[];
}

// POST /api/decide
// Make execution decision for a proposal
interface DecisionRequest {
  proposalId: number;
  proposal: Proposal;
  relevantOpinions: Opinion[];
}

interface DecisionResponse {
  decisionId: number;
  execute: boolean;
  reasoning: string;
  confidence: number;
  publishedToFeed: boolean; // Published to decision feed
}

// GET /api/decisions/feed
// Get decision feed (like Twitter timeline)
interface DecisionFeedEntry {
  decisionId: number;
  decision: string;
  reasoning: string;
  timestamp: number;
  relatedOpinions: number[];
  relatedProposals: number[];
  goodVotes: bigint;
  badVotes: bigint;
  communitySentiment: 'positive' | 'negative' | 'neutral';
}

// POST /api/decisions/vote
// Vote on a decision (good/bad)
interface DecisionVoteRequest {
  decisionId: number;
  isGood: boolean;
  amount: bigint;
}
```

**Implementation Example (Node.js):**

```typescript
import { OpenAI } from 'openai';
import { OpinionStaking } from './contracts';
import { TEEService } from '@jeju/council/tee';
import { ResearchAgent } from '@jeju/council/research-agent';
import { CEOAgent } from '@jeju/council/contracts';

class AICEOService {
  private openai: OpenAI;
  private opinionStaking: OpinionStaking;
  private decisionFeed: AICEDecisionFeed;
  private teeService: TEEService; // Council app TEE service
  private researchAgent: ResearchAgent; // Council app research agent
  private ceoAgent: CEOAgent; // Council app CEO agent contract
  private previousDecisions: DecisionFeedEntry[]; // For pattern detection
  
  async synthesizeOpinions(opinions: Opinion[]): Promise<SynthesisResponse> {
    // Cluster similar opinions
    const clusters = await this.clusterOpinions(opinions);
    
    // Get protocol state
    const protocolState = await this.getProtocolState();
    
    // Use ResearchAgent for deep analysis (Council app integration)
    const deepAnalysis = await this.researchAgent.deepAnalysis({
      opinions: opinions,
      clusters: clusters,
      protocolState: protocolState,
      useCompute: process.env.COMPUTE_ENABLED === 'true', // Use Compute marketplace if enabled
      model: process.env.COMPUTE_MODEL || 'claude-3-opus'
    });
    
    // Build prompt for AI (using deep analysis)
    const prompt = this.buildSynthesisPrompt(clusters, protocolState, deepAnalysis);
    
    // Call AI model
    const response = await this.openai.chat.completions.create({
      model: "gpt-4",
      messages: [
        {
          role: "system",
          content: "You are the AI CEO of Babylon/Jeju governance. Synthesize token holder opinions into actionable proposals."
        },
        {
          role: "user",
          content: prompt
        }
      ]
    });
    
    // Parse response into structured format
    return this.parseSynthesisResponse(response);
  }
  
  async makeDecision(
    proposalId: number,
    relevantOpinions: Opinion[]
  ): Promise<DecisionResponse> {
    // Synthesize opinions and make decision
    const synthesis = await this.synthesizeOpinions(relevantOpinions);
    
    // Make decision with TEE attestation (Council app integration)
    const decision = await this.teeService.makeAttestedDecision({
      reasoning: synthesis.reasoning,
      action: synthesis.action,
      proposalId: proposalId,
      timestamp: Date.now()
    });
    
    // Publish to decision feed
    const decisionId = await this.decisionFeed.publishDecision({
      decision: decision.action,
      reasoning: decision.reasoning,
      relatedOpinions: relevantOpinions.map(o => o.id),
      attestation: decision.attestation // TEE attestation proof
    });
    
    return {
      decisionId,
      execute: decision.execute,
      reasoning: decision.reasoning,
      confidence: decision.confidence,
      publishedToFeed: true,
      attestation: decision.attestation // Hardware attestation proof
    };
  }
  
  private async clusterOpinions(opinions: Opinion[]): Promise<Cluster[]> {
    // Use embeddings to cluster similar opinions
    // Implementation: OpenAI embeddings API or similar
    const embeddings = await this.getEmbeddings(opinions.map(o => o.text));
    return this.kMeansCluster(embeddings, opinions);
  }
  
  private buildSynthesisPrompt(clusters: Cluster[], state: ProtocolState): string {
    return `
Token Holder Opinions (weighted by stake):
${clusters.map(c => `- ${c.text} (Weight: ${c.totalWeight})`).join('\n')}

Current Protocol State:
- Active Users: ${state.activeUsers}
- Treasury: ${state.treasury}
- Recent Changes: ${state.recentChanges}

Synthesize these opinions into:
1. Prioritized list of actions
2. Specific proposals for specialized agents
3. Reasoning for each decision
`;
  }
}
```

### 2.2 Specialized Agent Framework

**Agent Interface:**

```typescript
interface SpecializedAgent {
  agentId: string; // EIP-8004 identity
  specialization: string; // Free-form specialization (market-driven, not enum)
  
  // Generate work based on input (compete with other agents)
  generateWork(input: AgentInput): Promise<AgentWork>;
  
  // Work can be: PR, Proposal, or Implementation
  // Agents compete - best work wins
  
  // Get agent performance metrics
  getPerformance(): Promise<PerformanceMetrics>;
}

interface AgentInput {
  opinions: Opinion[];
  protocolState: ProtocolState;
  context: any;
}

interface AgentWork {
  id: string;
  workType: 'pr' | 'proposal' | 'implementation';
  agentId: string; // EIP-8004 agent identity
  specialization: string;
  relatedOpinions: string[]; // Opinion IDs this work addresses
  
  // For PRs
  prNumber?: number;
  title?: string;
  description?: string;
  branch?: string;
  commits?: string[];
  filesChanged?: string[];
  testResults?: TestResults;
  
  // For Proposals
  proposalDescription?: string;
  executionData?: string;
  impactAnalysis?: string;
  costEstimate?: bigint;
  implementationPlan?: string;
  
  // For Direct Implementations
  implementationDetails?: string;
  codeChanges?: string;
  
  status: 'submitted' | 'review' | 'approved' | 'merged' | 'rejected' | 'competing';
  createdAt: number;
  mergedAt?: number;
  qualityScore?: number; // For competition evaluation
}
```

**Example: Technical Agent (Competing Work Generation)**

```typescript
class TechnicalAgent implements SpecializedAgent {
  agentId: string;
  specialization = 'technical'; // Free-form specialization
  private githubClient: GitHubClient;
  
  async generateWork(input: AgentInput): Promise<AgentWork> {
    // Analyze the task
    const task = await this.analyzeTask(input);
    
    // Generate work based on complexity
    if (task.complexity === 'simple') {
      // Generate PR directly
      return await this.generatePR(input);
    } else if (task.complexity === 'medium') {
      // Generate implementation
      return await this.generateImplementation(input);
    } else {
      // Generate proposal
      return await this.generateProposal(input);
    }
  }
  
  async generatePR(input: AgentInput): Promise<AgentWork> {
    // Analyze opinions to understand what needs to be fixed
    const technicalOpinions = input.opinions.filter(
      o => this.isTechnical(o.text)
    );
    
    // Implement the fix
    const implementation = await this.implementFix(technicalOpinions);
    
    // Write tests
    const tests = await this.writeTests(implementation);
    
    // Create branch and commit
    const branch = await this.githubClient.createBranch(`agent-${this.agentId}-${Date.now()}`);
    await this.githubClient.commit(branch, implementation, tests);
    
    // Submit PR
    const pr = await this.githubClient.createPR({
      title: this.generatePRTitle(technicalOpinions),
      description: this.generatePRDescription(technicalOpinions, implementation),
      branch: branch,
      base: 'main'
    });
    
    // Register work on-chain for deep funding tracking
    await this.registerWork(work);
    
    return {
      id: generateId(),
      workType: 'pr',
      agentId: this.agentId,
      specialization: this.specialization,
      relatedOpinions: technicalOpinions.map(o => o.id),
      prNumber: pr.number,
      title: pr.title,
      description: pr.body,
      branch: branch,
      commits: pr.commits,
      filesChanged: pr.files,
      testResults: await this.runTests(tests),
      impactAnalysis: await this.analyzeImpact(implementation),
      status: 'submitted',
      createdAt: Date.now(),
      qualityScore: await this.calculateQualityScore(pr, tests, implementation)
    };
  }
  
  async generateImplementation(input: AgentInput): Promise<AgentWork> {
    // Generate direct implementation (not PR, not proposal)
    const implementation = await this.implement(input);
    
    return {
      id: generateId(),
      workType: 'implementation',
      agentId: this.agentId,
      specialization: this.specialization,
      relatedOpinions: input.opinions.map(o => o.id),
      implementationDetails: implementation.details,
      codeChanges: implementation.code,
      status: 'submitted',
      createdAt: Date.now(),
      qualityScore: await this.calculateQualityScore(implementation)
    };
  }
  
  async generateProposal(input: AgentInput): Promise<AgentWork> {
    // Analyze technical opinions
    const technicalOpinions = input.opinions.filter(
      o => this.isTechnical(o.text)
    );
    
    // Analyze codebase and protocol state
    const codebaseAnalysis = await this.analyzeCodebase();
    const protocolMetrics = await this.getProtocolMetrics();
    
    // Generate technical proposal
    const proposal = await this.llm.generate({
      prompt: this.buildTechnicalPrompt(
        technicalOpinions,
        codebaseAnalysis,
        protocolMetrics
      ),
      model: "gpt-4"
    });
    
    return {
      id: generateId(),
      workType: 'proposal',
      agentId: this.agentId,
      specialization: this.specialization,
      relatedOpinions: technicalOpinions.map(o => o.id),
      proposalDescription: proposal.description,
      executionData: this.encodeExecutionData(proposal),
      impactAnalysis: proposal.impact,
      costEstimate: proposal.cost,
      implementationPlan: proposal.plan,
      status: 'submitted',
      createdAt: Date.now(),
      qualityScore: await this.calculateQualityScore(proposal)
    };
  }
  
  private async calculateQualityScore(work: any): Promise<number> {
    // Calculate quality score based on:
    // - Code quality (if PR)
    // - Test coverage
    // - Impact analysis
    // - Alignment with opinions
    // Returns score 0-100
    return 85; // Example
  }
  
  private buildTechnicalPrompt(
    opinions: Opinion[],
    codebase: CodebaseAnalysis,
    metrics: ProtocolMetrics
  ): string {
    return `
Technical Opinions:
${opinions.map(o => `- ${o.text} (Weight: ${o.weight})`).join('\n')}

Current Codebase:
${codebase.summary}

Protocol Metrics:
- Block Time: ${metrics.blockTime}ms
- Gas Usage: ${metrics.gasUsage}
- Throughput: ${metrics.throughput}

Generate a technical proposal that addresses these opinions.
Include: description, implementation plan, impact analysis, cost estimate.
`;
  }
}
```

### 2.3 Opinion Clustering Service

```typescript
class OpinionClusteringService {
  async clusterOpinions(opinions: Opinion[]): Promise<Cluster[]> {
    // Get embeddings for all opinions
    const embeddings = await this.getEmbeddings(
      opinions.map(o => o.text)
    );
    
    // Cluster using similarity
    const clusters: Cluster[] = [];
    const similarityThreshold = 0.85;
    
    for (let i = 0; i < opinions.length; i++) {
      let assigned = false;
      
      for (const cluster of clusters) {
        const similarity = this.cosineSimilarity(
          embeddings[i],
          cluster.centroid
        );
        
        if (similarity > similarityThreshold) {
          cluster.opinions.push(opinions[i]);
          cluster.totalWeight += opinions[i].weight;
          cluster.centroid = this.updateCentroid(cluster, embeddings[i]);
          assigned = true;
          break;
        }
      }
      
      if (!assigned) {
        clusters.push({
          id: generateId(),
          opinions: [opinions[i]],
          totalWeight: opinions[i].weight,
          centroid: embeddings[i],
          representativeText: opinions[i].text
        });
      }
    }
    
    return clusters;
  }
  
  private async getEmbeddings(texts: string[]): Promise<number[][]> {
    // Use OpenAI embeddings API or similar
    const response = await openai.embeddings.create({
      model: "text-embedding-3-large",
      input: texts
    });
    
    return response.data.map(d => d.embedding);
  }
  
  private cosineSimilarity(a: number[], b: number[]): number {
    let dotProduct = 0;
    let normA = 0;
    let normB = 0;
    
    for (let i = 0; i < a.length; i++) {
      dotProduct += a[i] * b[i];
      normA += a[i] * a[i];
      normB += b[i] * b[i];
    }
    
    return dotProduct / (Math.sqrt(normA) * Math.sqrt(normB));
  }
}
```

### 2.4 Work Review Process (Open Source Best Practices)

**Architecture:**
- **Specialized Review Agents:** Code review, security review, test review, alignment review agents (unlimited specializations)
- **Multi-Agent Review:** Multiple agents review each work item (like open source community)
- **Human-in-the-Loop (Initially):** Humans review agent recommendations before merge/execution
- **AI CEO Coordination:** AI CEO ensures humans only review meaningful items
- **Review Feed:** All reviews published transparently (like GitHub PR reviews)
- **Future:** As system matures, reduce human involvement (agents become trusted)

**Review Agents (Examples - Not Limited):**
- Code Review Agent: Reviews code quality, best practices, maintainability
- Security Review Agent: Reviews for vulnerabilities, security issues
- Test Review Agent: Reviews test coverage, test quality
- Alignment Review Agent: Reviews alignment with staked opinions
- And more specializations as market demands

**Review Process:**
1. Agent submits work (PR, proposal, or implementation)
2. Multiple review agents review work in parallel
3. Reviews published to review feed
4. AI CEO aggregates reviews and makes recommendation
5. If competing work: AI CEO evaluates all, selects best
6. Human reviews if needed (initially required for high-stakes, later optional)
7. Merge/execution decision based on review consensus

### 2.5 Work Review Orchestrator Service

```typescript
import { QualityOracle } from '@jeju/council/contracts';
import { ProposalAssist } from '@jeju/council/proposal-assistant';
import { CouncilAgents } from '@jeju/council/agents';

class WorkReviewOrchestrator {
  private reviewAgents: ReviewAgent[]; // Unlimited specializations
  private aiCEO: AICEOService;
  private githubClient: GitHubClient;
  private workTracking: WorkTracking;
  private qualityOracle: QualityOracle; // Council app integration
  private proposalAssist: ProposalAssist; // Council app integration
  private councilAgents: CouncilAgents; // Council app integration
  
  constructor() {
    // Initialize specialized review agents
    // Agents can be added dynamically based on market demand
    this.reviewAgents = [
      new CodeReviewAgent(),
      new SecurityReviewAgent(),
      new TestReviewAgent(),
      new AlignmentReviewAgent(),
      // CouncilAgents can also participate in reviews
      this.councilAgents.getSecurityAgent(),
      this.councilAgents.getCodeAgent()
      // More review agents can be added as needed
    ];
  }
  
  async addReviewAgent(agent: ReviewAgent): Promise<void> {
    // Dynamically add review agents based on market demand
    this.reviewAgents.push(agent);
  }
  
  async reviewWork(workId: string): Promise<WorkReviewResult> {
    // Get work details (PR, proposal, or implementation)
    const work = await this.getWork(workId);
    
    // Get related opinions
    const relatedOpinions = await this.getRelatedOpinions(work);
    
    // Parallel review by specialized agents
    const reviews = await Promise.all(
      this.reviewAgents.map(agent => agent.review(work))
    );
    
    // Use ProposalAssist for quality scoring (Council app integration)
    const proposalAssistScore = await this.proposalAssist.assessQuality({
      work: work,
      reviews: reviews,
      relatedOpinions: relatedOpinions
    });
    
    // Get quality score from QualityOracle (Council app integration)
    const qualityScore = await this.qualityOracle.assessQuality({
      workType: work.workType,
      agent: work.agent,
      reviews: reviews,
      proposalAssistScore: proposalAssistScore
    });
    
    // Publish reviews to feed
    await this.publishReviews(workId, reviews);
    
    // AI CEO aggregates reviews and makes recommendation
    const aiCEORecommendation = await this.aiCEO.aggregateReviews(
      reviews, 
      work, 
      relatedOpinions,
      qualityScore // Include quality score from oracle
    );
    
    // Determine if human review needed
    const needsHumanReview = this.shouldRequireHumanReview(
      reviews, 
      aiCEORecommendation, 
      work,
      qualityScore
    );
    
    return {
      workId,
      workType: work.workType,
      reviews: reviews,
      qualityScore: qualityScore, // From QualityOracle
      proposalAssistScore: proposalAssistScore, // From ProposalAssist
      aiCEORecommendation: aiCEORecommendation,
      needsHumanReview: needsHumanReview,
      approved: !needsHumanReview && aiCEORecommendation.approved && qualityScore >= MIN_QUALITY_SCORE,
      reasoning: aiCEORecommendation.reasoning
    };
  }
  
  async evaluateCompetingWork(workList: AgentWork[]): Promise<AgentWork> {
    // Evaluate multiple competing works
    // Score each work based on:
    // - Quality (code, tests, documentation)
    // - Alignment with staked opinions
    // - Impact analysis
    // - Cost/benefit ratio
    
    const scores = await Promise.all(
      workList.map(async (work) => {
        const review = await this.reviewWork(work.id);
        return {
          work,
          score: this.calculateWorkScore(work, review)
        };
      })
    );
    
    // Select best work
    const bestWork = scores.reduce((best, current) => 
      current.score > best.score ? current : best
    );
    
    return bestWork.work;
  }
  
  private shouldRequireHumanReview(
    reviews: Review[],
    aiCEORecommendation: AICEORecommendation,
    pr: PullRequest
  ): boolean {
    // AI CEO + specialized coding agents determine human review need
    
    // NO human review needed for:
    // - Small changes, bug fixes
    // - No security concerns
    // - Standard operations
    
    // Human review REQUIRED for:
    // - Big changes (significant code changes)
    // - Security concerns (security issues found)
    // - Protocol-impacting changes
    
    const hasSecurityIssues = reviews.some(r => r.type === 'security' && r.issues.length > 0);
    const isBigChange = pr.filesChanged.length > 10 || pr.commits.length > 5;
    const isProtocolImpact = aiCEORecommendation.impact === 'high' || aiCEORecommendation.impact === 'critical';
    
    // Small bug fixes don't need human review
    const isSmallBugFix = pr.filesChanged.length <= 3 && 
                          pr.commits.length <= 2 && 
                          !hasSecurityIssues &&
                          aiCEORecommendation.impact === 'low';
    
    if (isSmallBugFix) {
      return false; // No human review needed
    }
    
    // Human review required for big changes or security concerns
    return hasSecurityIssues || (isBigChange && isProtocolImpact);
  }
  
  async mergeWork(workId: string, reviewResult: WorkReviewResult, humanApproval?: boolean): Promise<void> {
    // Require human approval if needed
    if (reviewResult.needsHumanReview && !humanApproval) {
      throw new Error("Human approval required");
    }
    
    if (!reviewResult.approved && !humanApproval) {
      throw new Error("Work not approved");
    }
    
    const work = await this.getWork(workId);
    
    // Trusted developers can merge directly (transitional phase)
    const isTrustedDeveloper = await this.isTrustedDeveloper(msg.sender);
    
    if (work.workType === 'pr') {
      // Merge PR on GitHub
      if (isTrustedDeveloper && reviewResult.approved) {
        await this.githubClient.mergePR(work.prNumber!);
      } else {
        await this.githubClient.mergePR(work.prNumber!);
      }
      
      // Mark PR as merged on-chain
      await this.workTracking.markWorkMerged(work.id);
    } else if (work.workType === 'proposal') {
      // Execute proposal
      await this.executeProposal(work);
    } else if (work.workType === 'implementation') {
      // Apply implementation
      await this.applyImplementation(work);
    }
    
    // Assign credit for deep funding
    const credit = this.calculateWorkCredit(reviewResult, work);
    await this.workTracking.assignWorkCredit(workId, credit);
    
    // Register contribution for deep funding
    await this.deepFunding.registerWorkContribution(work.agentId, workId, credit);
  }
  
  private async isTrustedDeveloper(address: string): Promise<boolean> {
    return await this.trustedDevelopersRegistry.isTrusted(address);
  }
  
  async mergePR(prNumber: number, reviewResult: PRReviewResult): Promise<void> {
    if (!reviewResult.approved) {
      throw new Error("PR not approved for merge");
    }
    
    // Merge PR on GitHub
    await this.githubClient.mergePR(prNumber);
    
    // Mark work as merged on-chain
    await this.workTracking.markWorkMerged(work.id);
    
    // Assign credit for deep funding
    const credit = this.calculateWorkCredit(reviewResult, work);
    await this.workTracking.assignWorkCredit(work.id, credit);
    
    // Register contribution for deep funding
    await this.deepFunding.registerWorkContribution(work.agentId, work.id, credit);
  }
  
  private async reviewCode(pr: PullRequest): Promise<CodeReview> {
    const prompt = `
Review this pull request for code quality:

Title: ${pr.title}
Description: ${pr.description}
Files Changed: ${pr.filesChanged.join(', ')}
Commits: ${pr.commits.join(', ')}

Evaluate:
1. Code quality and best practices
2. Potential bugs or security issues
3. Performance implications
4. Alignment with project standards

Provide a score (0-100) and detailed feedback.
`;
    
    const response = await this.openai.chat.completions.create({
      model: "gpt-4",
      messages: [
        { role: "system", content: "You are a code reviewer for Babylon/Jeju projects." },
        { role: "user", content: prompt }
      ]
    });
    
    return this.parseCodeReview(response);
  }
  
  private calculatePRCredit(reviewResult: PRReviewResult): number {
    // Credit based on:
    // - Code quality (40%)
    // - Test coverage (30%)
    // - Impact (20%)
    // - Alignment with opinions (10%)
    
    const codeQualityScore = reviewResult.codeQuality / 100;
    const testCoverageScore = reviewResult.testCoverage / 100;
    const impactScore = reviewResult.impact / 100;
    const alignmentScore = 1.0; // Assume aligned if approved
    
    const totalScore = (
      codeQualityScore * 0.4 +
      testCoverageScore * 0.3 +
      impactScore * 0.2 +
      alignmentScore * 0.1
    );
    
    // Convert to basis points (0-10000)
    return Math.floor(totalScore * 10000);
  }
}
```

### 2.5 Deep Funding Service

```typescript
class DeepFundingService {
  async calculateCreditAssignment(
    contributions: Contribution[],
    juryEvaluations: JuryEvaluation[]
  ): Promise<CreditAssignment> {
    // Include work contributions (PRs, proposals, implementations)
    const workContributions = await this.getWorkContributions();
    const allContributions = [...contributions, ...workContributions];
    
    // Get AI agent full lists
    const agentFullLists = await this.getAgentFullLists(allContributions);
    
    // Find best-fit combination using linear regression
    const bestFit = this.findBestFitCombination(
      agentFullLists,
      juryEvaluations
    );
    
    return {
      coefficients: bestFit.coefficients,
      finalAssignment: bestFit.assignment
    };
  }
  
  async getWorkContributions(): Promise<Contribution[]> {
    // Get all merged works
    const allWorks = await this.workTracking.getAllMergedWorks();
    
    return allWorks.map(work => ({
      contributor: work.agent,
      description: work.workType == 0 
        ? `PR #${work.prNumber}: ${work.title}`
        : (work.workType == 1 ? `Proposal: ${work.title}` : `Implementation: ${work.title}`),
      credit: work.credit,
      contributionType: 'Code'
    }));
  }
  
  private findBestFitCombination(
    agentLists: AgentFullList[],
    juryEvaluations: JuryEvaluation[]
  ): BestFit {
    // Use least squares to find linear combination
    // that best matches jury evaluations
    // Implementation: scikit-learn or similar
    
    const X = this.buildMatrix(agentLists, juryEvaluations);
    const y = juryEvaluations.map(e => e.credit);
    
    // Solve: min ||X * coefficients - y||^2
    const coefficients = this.leastSquares(X, y);
    
    // Generate final assignment
    const assignment = this.combineAgentLists(
      agentLists,
      coefficients
    );
    
    return { coefficients, assignment };
  }
  
  async buildDependencyGraph(
    rootNode: string
  ): Promise<DependencyGraph> {
    // Recursively build dependency graph
    const graph: DependencyGraph = {
      node: rootNode,
      children: []
    };
    
    const dependencies = await this.getDependencies(rootNode);
    
    for (const dep of dependencies) {
      const childGraph = await this.buildDependencyGraph(dep.address);
      childGraph.weight = dep.weight;
      graph.children.push(childGraph);
    }
    
    return graph;
  }
}
```

### 2.6 Execution Simulation Service

**Purpose:** Simulate proposal execution before actual execution to detect issues, estimate gas, and verify correctness.

**Architecture:**
- Sandbox environment (fork of Jeju Network)
- State snapshot and restoration
- Gas estimation
- Side effect detection
- Error detection and reporting

**Implementation:**

```typescript
import { ethers } from 'ethers';
import { JsonRpcProvider } from '@ethersproject/providers';

interface SimulationResult {
  success: boolean;
  gasEstimate: number;
  stateChanges: StateChange[];
  sideEffects: SideEffect[];
  warnings: string[];
  errors: string[];
  executionLog: string[];
}

interface StateChange {
  contract: string;
  slot: string;
  oldValue: string;
  newValue: string;
}

interface SideEffect {
  type: 'transfer' | 'event' | 'call' | 'deployment';
  description: string;
  impact: 'low' | 'medium' | 'high';
}

class ExecutionSimulator {
  private provider: JsonRpcProvider;
  private sandboxRpc: string; // Sandbox/forked network RPC
  
  constructor(sandboxRpc: string) {
    this.sandboxRpc = sandboxRpc;
    this.provider = new JsonRpcProvider(sandboxRpc);
  }
  
  async simulateExecution(proposal: Proposal): Promise<SimulationResult> {
    // 1. Create sandbox snapshot
    const snapshotId = await this.createSnapshot();
    
    try {
      // 2. Load current protocol state
      const currentState = await this.loadProtocolState();
      
      // 3. Execute proposal in sandbox
      const executionResult = await this.executeInSandbox(proposal, currentState);
      
      // 4. Analyze results
      const analysis = await this.analyzeExecution(executionResult);
      
      // 5. Restore snapshot
      await this.restoreSnapshot(snapshotId);
      
      return {
        success: executionResult.success,
        gasEstimate: executionResult.gasUsed,
        stateChanges: analysis.stateChanges,
        sideEffects: analysis.sideEffects,
        warnings: analysis.warnings,
        errors: analysis.errors,
        executionLog: executionResult.logs
      };
    } catch (error) {
      // Restore snapshot on error
      await this.restoreSnapshot(snapshotId);
      throw error;
    }
  }
  
  private async createSnapshot(): Promise<string> {
    // Create snapshot of current sandbox state
    // Using hardhat_evm_snapshot or similar
    const response = await fetch(this.sandboxRpc, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method: 'evm_snapshot',
        params: [],
        id: 1
      })
    });
    
    const data = await response.json();
    return data.result;
  }
  
  private async restoreSnapshot(snapshotId: string): Promise<void> {
    // Restore sandbox state
    await fetch(this.sandboxRpc, {
      method: 'POST',
      headers: { 'Content-Type': 'application/json' },
      body: JSON.stringify({
        jsonrpc: '2.0',
        method: 'evm_revert',
        params: [snapshotId],
        id: 1
      })
    });
  }
  
  private async loadProtocolState(): Promise<ProtocolState> {
    // Load current protocol state from sandbox
    const state = {
      blockNumber: await this.provider.getBlockNumber(),
      balances: {},
      contractStates: {},
      // ... other state
    };
    
    // Load balances of relevant addresses
    // Load contract states
    // Load storage slots
    
    return state;
  }
  
  private async executeInSandbox(
    proposal: Proposal,
    state: ProtocolState
  ): Promise<ExecutionResult> {
    // Execute proposal in sandbox environment
    const tx = {
      to: proposal.targetContract,
      data: proposal.executionData,
      gasLimit: 10000000, // High gas limit for simulation
    };
    
    try {
      const result = await this.provider.call(tx, state.blockNumber);
      const receipt = await this.provider.getTransactionReceipt(result.hash);
      
      return {
        success: receipt.status === 1,
        gasUsed: receipt.gasUsed.toNumber(),
        logs: receipt.logs,
        returnValue: result
      };
    } catch (error) {
      return {
        success: false,
        gasUsed: 0,
        logs: [],
        returnValue: null,
        error: error.message
      };
    }
  }
  
  private async analyzeExecution(
    result: ExecutionResult
  ): Promise<ExecutionAnalysis> {
    const analysis: ExecutionAnalysis = {
      stateChanges: [],
      sideEffects: [],
      warnings: [],
      errors: []
    };
    
    // Analyze state changes
    analysis.stateChanges = await this.detectStateChanges(result);
    
    // Detect side effects
    analysis.sideEffects = await this.detectSideEffects(result);
    
    // Check for warnings
    analysis.warnings = await this.checkWarnings(result);
    
    // Check for errors
    if (!result.success) {
      analysis.errors.push(result.error || 'Execution failed');
    }
    
    // Check gas usage
    if (result.gasUsed > 5000000) {
      analysis.warnings.push('High gas usage detected');
    }
    
    // Check for unexpected transfers
    const transfers = await this.detectTransfers(result);
    for (const transfer of transfers) {
      if (transfer.amount > 100000e18) {
        analysis.warnings.push(`Large transfer detected: ${transfer.amount} to ${transfer.to}`);
      }
    }
    
    return analysis;
  }
  
  private async detectStateChanges(result: ExecutionResult): Promise<StateChange[]> {
    // Compare state before and after execution
    // This would require tracking storage slots
    // Simplified version:
    return []; // Would implement full state diff
  }
  
  private async detectSideEffects(result: ExecutionResult): Promise<SideEffect[]> {
    const sideEffects: SideEffect[] = [];
    
    // Analyze logs for events
    for (const log of result.logs) {
      // Parse event
      // Check if it's expected or unexpected
      sideEffects.push({
        type: 'event',
        description: `Event emitted: ${log.topics[0]}`,
        impact: 'low'
      });
    }
    
    return sideEffects;
  }
  
  private async checkWarnings(result: ExecutionResult): Promise<string[]> {
    const warnings: string[] = [];
    
    // Check for common issues
    if (result.gasUsed > 3000000) {
      warnings.push('High gas usage');
    }
    
    // Check for reentrancy patterns
    // Check for access control issues
    // Check for overflow/underflow
    
    return warnings;
  }
  
  private async detectTransfers(result: ExecutionResult): Promise<Transfer[]> {
    // Parse logs for Transfer events
    // Return list of transfers
    return [];
  }
}

// Usage
const simulator = new ExecutionSimulator('http://localhost:8545');
const result = await simulator.simulateExecution(proposal);

if (!result.success) {
  console.error('Simulation failed:', result.errors);
} else {
  console.log('Gas estimate:', result.gasEstimate);
  console.log('Warnings:', result.warnings);
}
```

### 2.7 Financial Transaction Classification Service

**Purpose:** Classify financial transactions to determine if they require human review or can be executed automatically.

**Implementation:**

```typescript
interface FinancialTransaction {
  recipient: string;
  amount: bigint;
  token: string;
  timestamp: number;
  proposalId: number;
}

interface TransactionClassification {
  isStandardPayment: boolean;
  isKnownAddress: boolean;
  isRegularPayment: boolean;
  isUnusualPayment: boolean;
  isBigPayment: boolean;
  isNewPayment: boolean;
  requiresVeto: boolean;
  riskLevel: 'low' | 'medium' | 'high';
  reasoning: string;
}

class FinancialTransactionClassifier {
  private knownAddressRegistry: KnownAddressRegistry;
  private paymentHistory: Map<string, PaymentHistory>;
  
  constructor(knownAddressRegistry: KnownAddressRegistry) {
    this.knownAddressRegistry = knownAddressRegistry;
    this.paymentHistory = new Map();
  }
  
  async classifyTransaction(
    tx: FinancialTransaction
  ): Promise<TransactionClassification> {
    // Check if address is known
    const isKnownAddress = await this.knownAddressRegistry.isKnownAddress(tx.recipient);
    
    // Check payment history
    const history = await this.getPaymentHistory(tx.recipient);
    
    // Check if it's a regular payment
    const isRegularPayment = this.isRegularPayment(tx, history);
    
    // Check if it's a big payment
    const isBigPayment = tx.amount > BigInt(100000e18); // 100K tokens
    
    // Check if it's a new payment (first time to this address)
    const isNewPayment = history.paymentCount === 0;
    
    // Check if it's unusual
    const isUnusualPayment = this.isUnusualPayment(tx, history);
    
    // Determine if standard payment
    const isStandardPayment = isKnownAddress && 
                              isRegularPayment && 
                              !isBigPayment && 
                              !isUnusualPayment;
    
    // Determine if requires veto
    const requiresVeto = isBigPayment || isUnusualPayment || isNewPayment || !isKnownAddress;
    
    // Calculate risk level
    const riskLevel = this.calculateRiskLevel(tx, {
      isKnownAddress,
      isRegularPayment,
      isBigPayment,
      isUnusualPayment,
      isNewPayment
    });
    
    // Generate reasoning
    const reasoning = this.generateReasoning(tx, {
      isStandardPayment,
      isKnownAddress,
      isRegularPayment,
      isBigPayment,
      isUnusualPayment,
      isNewPayment,
      requiresVeto,
      riskLevel
    });
    
    return {
      isStandardPayment,
      isKnownAddress,
      isRegularPayment,
      isUnusualPayment,
      isBigPayment,
      isNewPayment,
      requiresVeto,
      riskLevel,
      reasoning
    };
  }
  
  private async getPaymentHistory(recipient: string): Promise<PaymentHistory> {
    if (this.paymentHistory.has(recipient)) {
      return this.paymentHistory.get(recipient)!;
    }
    
    // Load from on-chain registry
    const history = await this.knownAddressRegistry.getPaymentHistory(recipient);
    this.paymentHistory.set(recipient, history);
    
    return history;
  }
  
  private isRegularPayment(
    tx: FinancialTransaction,
    history: PaymentHistory
  ): boolean {
    // Regular payment if:
    // - Has received at least 3 payments
    // - Last payment was within 30 days
    // - Amount is similar to previous payments (±50%)
    
    if (history.paymentCount < 3) return false;
    
    const daysSinceLastPayment = (Date.now() - history.lastPayment) / (1000 * 60 * 60 * 24);
    if (daysSinceLastPayment > 30) return false;
    
    const avgAmount = history.totalAmount / BigInt(history.paymentCount);
    const amountDiff = tx.amount > avgAmount 
      ? tx.amount - avgAmount 
      : avgAmount - tx.amount;
    const percentDiff = Number(amountDiff * BigInt(100) / avgAmount);
    
    return percentDiff <= 50; // Within 50% of average
  }
  
  private isUnusualPayment(
    tx: FinancialTransaction,
    history: PaymentHistory
  ): boolean {
    if (history.paymentCount === 0) return true;
    
    // Unusual if:
    // - Amount is significantly different from average (>200% or <50%)
    // - Payment frequency is unusual
    // - Payment pattern changed
    
    const avgAmount = history.totalAmount / BigInt(history.paymentCount);
    const amountDiff = tx.amount > avgAmount 
      ? tx.amount - avgAmount 
      : avgAmount - tx.amount;
    const percentDiff = Number(amountDiff * BigInt(100) / avgAmount);
    
    if (percentDiff > 200 || percentDiff < 50) return true;
    
    // Check frequency
    const avgInterval = history.totalInterval / history.paymentCount;
    const currentInterval = Date.now() - history.lastPayment;
    if (currentInterval < avgInterval / 2 || currentInterval > avgInterval * 2) {
      return true;
    }
    
    return false;
  }
  
  private calculateRiskLevel(
    tx: FinancialTransaction,
    flags: {
      isKnownAddress: boolean;
      isRegularPayment: boolean;
      isBigPayment: boolean;
      isUnusualPayment: boolean;
      isNewPayment: boolean;
    }
  ): 'low' | 'medium' | 'high' {
    if (flags.isKnownAddress && flags.isRegularPayment && !flags.isBigPayment) {
      return 'low';
    }
    
    if (flags.isBigPayment || flags.isNewPayment) {
      return 'high';
    }
    
    if (flags.isUnusualPayment || !flags.isKnownAddress) {
      return 'medium';
    }
    
    return 'low';
  }
  
  private generateReasoning(
    tx: FinancialTransaction,
    classification: TransactionClassification
  ): string {
    const reasons: string[] = [];
    
    if (classification.isStandardPayment) {
      reasons.push('Standard payment to known address');
    } else {
      if (!classification.isKnownAddress) {
        reasons.push('Unknown recipient address');
      }
      if (classification.isBigPayment) {
        reasons.push(`Large payment amount: ${tx.amount}`);
      }
      if (classification.isUnusualPayment) {
        reasons.push('Unusual payment pattern');
      }
      if (classification.isNewPayment) {
        reasons.push('First payment to this address');
      }
    }
    
    return reasons.join(', ');
  }
}

interface PaymentHistory {
  recipient: string;
  paymentCount: number;
  totalAmount: bigint;
  lastPayment: number;
  totalInterval: number; // Total time between payments
  averageAmount: bigint;
}
```

### 2.8 Trusted Registry Service

**Purpose:** Manage trusted developers and contributors with GitHub access and financial transaction controls.

**Implementation:**

```typescript
import { TrustedRegistry } from './contracts';

interface TrustedEntity {
  address: string;
  trustType: 'Developer' | 'Contributor';
  description: string;
  addedAt: number;
  active: boolean;
  githubUsername?: string; // For developers
}

class TrustedRegistryService {
  private contract: TrustedRegistry;
  private githubClient: GitHubClient;
  
  constructor(contractAddress: string, githubClient: GitHubClient) {
    this.contract = new TrustedRegistry(contractAddress);
    this.githubClient = githubClient;
  }
  
  async addTrustedDeveloper(
    address: string,
    description: string,
    githubUsername: string
  ): Promise<void> {
    // Add to on-chain registry
    await this.contract.addTrustedEntity(
      address,
      TrustType.Developer,
      description
    );
    
    // Grant GitHub access (if applicable)
    if (githubUsername) {
      await this.githubClient.addCollaborator(githubUsername, 'write'); // Write access for merges
    }
  }
  
  async addTrustedContributor(
    address: string,
    description: string
  ): Promise<void> {
    // Add to on-chain registry
    await this.contract.addTrustedEntity(
      address,
      TrustType.Contributor,
      description
    );
  }
  
  async removeTrustedEntity(address: string): Promise<void> {
    // Remove from on-chain registry
    await this.contract.removeTrustedEntity(address);
    
    // Revoke GitHub access if developer
    const entity = await this.contract.getTrustedEntity(address);
    if (entity.trustType === TrustType.Developer && entity.githubUsername) {
      await this.githubClient.removeCollaborator(entity.githubUsername);
    }
  }
  
  async isTrustedDeveloper(address: string): Promise<boolean> {
    return await this.contract.isTrusted(address, TrustType.Developer);
  }
  
  async isTrustedContributor(address: string): Promise<boolean> {
    return await this.contract.isTrusted(address, TrustType.Contributor);
  }
  
  async getTrustedDevelopers(): Promise<TrustedEntity[]> {
    const addresses = await this.contract.getTrustedDevelopers();
    const entities: TrustedEntity[] = [];
    
    for (const addr of addresses) {
      const entity = await this.contract.getTrustedEntity(addr);
      entities.push({
        address: addr,
        trustType: 'Developer',
        description: entity.description,
        addedAt: entity.addedAt.toNumber(),
        active: entity.active
      });
    }
    
    return entities;
  }
  
  async getTrustedContributors(): Promise<TrustedEntity[]> {
    const addresses = await this.contract.getTrustedContributors();
    const entities: TrustedEntity[] = [];
    
    for (const addr of addresses) {
      const entity = await this.contract.getTrustedEntity(addr);
      entities.push({
        address: addr,
        trustType: 'Contributor',
        description: entity.description,
        addedAt: entity.addedAt.toNumber(),
        active: entity.active
      });
    }
    
    return entities;
  }
}
```

---

## 3. DATA STORAGE & INDEXING

### 3.1 On-Chain Storage

- **Opinion Staking:** All stakes stored on-chain
- **Proposal Execution:** All executions recorded on-chain
- **PR Tracking:** All PRs registered and tracked on-chain
- **Deep Funding:** All distributions on-chain

### 3.2 Off-Chain Indexing (GraphQL)

```graphql
type Opinion {
  id: ID!
  text: String!
  creator: Address!
  totalStaked: BigInt!
  weight: BigInt!
  clusterId: ID
  timestamp: Int!
  stakes: [Stake!]!
}

type Stake {
  staker: Address!
  amount: BigInt!
  timestamp: Int!
}

type Proposal {
  id: ID!
  description: String!
  proposer: Address!
  agent: Agent!
  specialization: String!
  executionData: Bytes!
  executionTime: Int!
  executed: Boolean!
  vetoed: Boolean!
  relatedOpinions: [Opinion!]!
}

type Cluster {
  id: ID!
  opinions: [Opinion!]!
  totalWeight: BigInt!
  representativeText: String!
}

type Agent {
  address: Address!
  specialization: String! # Free-form, market-driven
  description: String!
  eip8004Reputation: Int!
  performanceScore: Int!
  prsMerged: Int!
  proposalsAccepted: Int!
  implementationsDelivered: Int!
  totalCredit: BigInt!
  earnings: BigInt!
  active: Boolean!
  verified: Boolean!
}

type AgentWork {
  id: ID!
  workType: WorkType! # PR, Proposal, or Implementation
  agent: Agent!
  specialization: String!
  relatedOpinions: [Opinion!]!
  status: WorkStatus!
  qualityScore: Float
  createdAt: Int!
  mergedAt: Int
  credit: BigInt!
  
  # PR fields
  prNumber: Int
  title: String
  repository: String
  description: String
  branch: String
  commits: [String!]
  filesChanged: [String!]
  testResults: TestResults
  
  # Proposal fields
  proposalDescription: String
  executionData: String
  impactAnalysis: String
  costEstimate: BigInt
  
  # Implementation fields
  implementationDetails: String
  codeChanges: String
}

enum WorkType {
  PR
  PROPOSAL
  IMPLEMENTATION
}

enum WorkStatus {
  SUBMITTED
  REVIEW
  APPROVED
  MERGED
  REJECTED
  COMPETING
}

enum PRStatus {
  OPEN
  REVIEW
  APPROVED
  MERGED
  REJECTED
}

type TestResults {
  passed: Int!
  failed: Int!
  coverage: Float!
  duration: Int!
}

type Query {
  opinions(
    filter: OpinionFilter
    orderBy: OpinionOrderBy
    limit: Int
  ): [Opinion!]!
  
  clusters: [Cluster!]!
  
  proposals(
    filter: ProposalFilter
  ): [Proposal!]!
  
  agents(
    specialization: String
  ): [Agent!]!
  
  agentWork(
    filter: WorkFilter
  ): [AgentWork!]!
  
  opinionWeight(opinionId: ID!): BigInt!
}
```

### 3.3 IPFS Storage

- Opinion text (full text stored on IPFS, hash on-chain)
- Proposal details and reasoning
- AI CEO decision logs
- Agent performance reports

---

## 4. INTEGRATION WITH EXISTING SYSTEMS

### 4.1 EIP-8004 Integration

```typescript
import { EIP8004Registry } from '@eip8004/registry';

class GovernanceEIP8004Integration {
  async registerAgent(agentAddress: string, metadata: AgentMetadata) {
    // Register agent with EIP-8004
    await EIP8004Registry.register({
      address: agentAddress,
      type: 'governance-agent',
      metadata: {
        specialization: metadata.specialization,
        description: metadata.description,
        performanceScore: metadata.performanceScore
      }
    });
  }
  
  async getAgentReputation(agentAddress: string): Promise<number> {
    // Get agent reputation from EIP-8004
    const attestations = await EIP8004Registry.getAttestations(agentAddress);
    return this.calculateReputation(attestations);
  }
}
```

### 4.2 Jeju Network Integration

```typescript
class JejuNetworkIntegration {
  async executeProposal(proposal: Proposal) {
    // Execute proposal on Jeju Network
    const tx = await jejuNetwork.sendTransaction({
      to: proposal.targetContract,
      data: proposal.executionData,
      gasLimit: proposal.gasEstimate
    });
    
    return tx.hash;
  }
  
  async getProtocolState(): Promise<ProtocolState> {
    // Query Jeju Network state
    return {
      blockTime: await jejuNetwork.getBlockTime(),
      gasPrice: await jejuNetwork.getGasPrice(),
      activeUsers: await this.getActiveUsers(),
      treasury: await this.getTreasuryBalance()
    };
  }
}
```

### 4.3 Gateway Integration

**Purpose:** Integrate governance with Gateway (bridge, paymasters, staking).

**Integration Points:**

1. **Paymaster Governance:**
   - Governance can decide paymaster parameters (fees, supported tokens)
   - Opinions can be about paymaster configurations
   - Agents can optimize paymaster settings

2. **Staking Integration:**
   - Governance token staking can use Gateway staking mechanisms
   - Gateway staking can be managed through governance

3. **Bridge Governance:**
   - Governance can decide bridge parameters (fees, supported chains)
   - Opinions can be about bridge configurations

**Implementation:**

```typescript
import { GatewayContract } from '@jeju/gateway';

class GatewayGovernanceIntegration {
  private gateway: GatewayContract;
  private opinionStaking: OpinionStaking;
  
  async createPaymasterOpinion(
    text: string,
    paymasterAddress: string,
    proposedFees: PaymasterFees
  ): Promise<number> {
    // Create app-scoped opinion for Gateway
    const opinionId = await opinionStaking.createOpinion(
      text,
      ipfsHash,
      OpinionScope.App,
      gateway.address,
      [gatewayToken, jejuToken] // Accept Gateway and Jeju tokens
    );
    
    return opinionId;
  }
  
  async executePaymasterProposal(
    proposalId: number,
    paymasterAddress: string,
    newFees: PaymasterFees
  ): Promise<void> {
    // Execute paymaster parameter change
    await gateway.updatePaymasterFees(paymasterAddress, newFees);
  }
  
  async getPaymasterState(): Promise<PaymasterState> {
    return {
      paymasters: await gateway.getPaymasters(),
      fees: await gateway.getPaymasterFees(),
      supportedTokens: await gateway.getSupportedTokens()
    };
  }
}
```

### 4.4 Crucible Integration

**Purpose:** Integrate governance with Crucible (agent orchestration) - shared agent registry.

**Integration Points:**

1. **Shared Agent Registry:**
   - Governance and Crucible share ERC-8004 agent registry
   - Agents registered in Crucible can participate in governance
   - Governance agents can be orchestrated by Crucible

2. **Agent Orchestration:**
   - Crucible can orchestrate governance agents
   - Governance agents can be part of Crucible workflows

3. **Cross-Pollination:**
   - Agents from Crucible can work on governance tasks
   - Governance agents can be used in Crucible orchestration

**Implementation:**

```typescript
import { CrucibleAgentRegistry } from '@jeju/crucible';
import { GovernanceAgentRegistry } from './AgentRegistry';

class CrucibleGovernanceIntegration {
  private crucibleRegistry: CrucibleAgentRegistry;
  private governanceRegistry: GovernanceAgentRegistry;
  private eip8004Registry: EIP8004Registry;
  
  async registerAgentForBoth(
    agentAddress: string,
    specialization: string,
    description: string
  ): Promise<void> {
    // Register in Crucible (for orchestration)
    await crucibleRegistry.registerAgent(agentAddress, {
      specialization,
      description,
      orchestrationEnabled: true
    });
    
    // Register in Governance (for governance work)
    await governanceRegistry.registerAgent(
      agentAddress,
      specialization,
      description
    );
    
    // Both use same ERC-8004 identity
    const reputation = await eip8004Registry.getReputation(agentAddress);
    // Reputation shared across both systems
  }
  
  async getAgentsForOrchestration(): Promise<Agent[]> {
    // Get agents that can be orchestrated by Crucible
    const crucibleAgents = await crucibleRegistry.getAgents();
    const governanceAgents = await governanceRegistry.getAgents();
    
    // Find agents registered in both
    return crucibleAgents.filter(agent => 
      governanceAgents.some(gAgent => gAgent.address === agent.address)
    );
  }
  
  async orchestrateGovernanceAgent(
    agentAddress: string,
    task: OrchestrationTask
  ): Promise<void> {
    // Crucible orchestrates governance agent
    await crucibleRegistry.orchestrateAgent(agentAddress, {
      ...task,
      context: 'governance'
    });
  }
}
```

### 4.6 Council App Integration

**Purpose:** Integrate with Council app's security and infrastructure features.

#### 4.6.1 TEE Service Integration
**Purpose:** Hardware-attested AI CEO decisions for security and verifiability.

**Integration:**
- AI CEO uses Council app's TEE Service (Phala Cloud) for decision-making
- Hardware attestation provides cryptographic proof of decision integrity
- Optional: Simulated TEE for development, hardware TEE for production
- Decisions include attestation proof for transparency

**Implementation:**
```typescript
import { TEEService } from '@jeju/council/tee';
import { CEOAgent } from '@jeju/council/contracts';

class AICEOService {
  private teeService: TEEService;
  private ceoAgent: CEOAgent;
  
  async makeDecision(opinions: Opinion[]): Promise<Decision> {
    // Synthesize opinions
    const synthesis = await this.synthesizeOpinions(opinions);
    
    // Make decision with TEE attestation
    const decision = await this.teeService.makeAttestedDecision({
      reasoning: synthesis.reasoning,
      action: synthesis.action,
      timestamp: Date.now()
    });
    
    // Publish to decision feed with attestation
    await this.decisionFeed.publishDecision({
      ...decision,
      attestation: decision.attestation // Hardware attestation proof
    });
    
    return decision;
  }
}
```

#### 4.6.2 QualityOracle Integration
**Purpose:** On-chain quality assessment for agent work.

**Integration:**
- QualityOracle.sol (from Council app) assesses agent work quality
- Provides on-chain quality scores for competition evaluation
- Used by review agents and AI CEO for work selection

**Implementation:**
```typescript
import { QualityOracle } from '@jeju/council/contracts';

class WorkReviewOrchestrator {
  private qualityOracle: QualityOracle;
  
  async reviewWork(work: AgentWork): Promise<WorkReviewResult> {
    // Review agents evaluate
    const reviews = await this.reviewAgents.review(work);
    
    // Get quality score from oracle
    const qualityScore = await this.qualityOracle.assessQuality({
      workType: work.workType,
      agent: work.agent,
      reviews: reviews
    });
    
    return {
      reviews,
      qualityScore, // On-chain quality score
      approved: qualityScore >= MIN_QUALITY_SCORE
    };
  }
}
```

#### 4.6.3 CouncilAgents Integration
**Purpose:** Leverage existing specialized agents from Council app.

**Integration:**
- CouncilAgents participate in agent marketplace:
  - TreasuryAgent: Treasury management
  - CodeAgent: Code review and implementation
  - CommunityAgent: Community engagement
  - SecurityAgent: Security reviews
- CouncilAgents can compete with other specialized agents
- Share ERC-8004 registry

**Implementation:**
```typescript
import { CouncilAgents } from '@jeju/council/agents';

class AgentRegistry {
  private councilAgents: CouncilAgents;
  
  async getAvailableAgents(): Promise<Agent[]> {
    // Get CouncilAgents
    const councilAgents = [
      this.councilAgents.getTreasuryAgent(),
      this.councilAgents.getCodeAgent(),
      this.councilAgents.getCommunityAgent(),
      this.councilAgents.getSecurityAgent()
    ];
    
    // Get other specialized agents
    const otherAgents = await this.getOtherAgents();
    
    // All agents can compete
    return [...councilAgents, ...otherAgents];
  }
}
```

#### 4.6.4 ProposalAssist Integration
**Purpose:** Quality scoring and attestation for agent-generated work.

**Integration:**
- ProposalAssist evaluates agent work (PRs, proposals, implementations)
- Provides quality scores for competition
- Attestation for work verification

**Implementation:**
```typescript
import { ProposalAssist } from '@jeju/council/proposal-assistant';

class WorkReviewOrchestrator {
  private proposalAssist: ProposalAssist;
  
  async evaluateWork(work: AgentWork): Promise<WorkEvaluation> {
    // ProposalAssist assesses quality
    const assessment = await this.proposalAssist.assessQuality({
      work: work,
      reviews: await this.reviewAgents.review(work)
    });
    
    return {
      qualityScore: assessment.score,
      attestation: assessment.attestation,
      recommendations: assessment.recommendations
    };
  }
}
```

#### 4.6.5 ResearchAgent Integration
**Purpose:** Deep analysis and research for AI CEO synthesis.

**Integration:**
- ResearchAgent assists AI CEO in opinion synthesis
- Uses Compute marketplace for deep analysis (when enabled)
- Falls back to local Ollama for standard analysis

**Implementation:**
```typescript
import { ResearchAgent } from '@jeju/council/research-agent';

class AICEOService {
  private researchAgent: ResearchAgent;
  
  async synthesizeOpinions(opinions: Opinion[]): Promise<SynthesisResponse> {
    // Use ResearchAgent for deep analysis
    const analysis = await this.researchAgent.deepAnalysis({
      opinions: opinions,
      useCompute: process.env.COMPUTE_ENABLED === 'true',
      model: process.env.COMPUTE_MODEL || 'claude-3-opus'
    });
    
    // Synthesize with deep analysis
    return this.synthesize(analysis);
  }
}
```

### 4.7 Compute Integration

**Purpose:** Integrate governance with Compute (AI inference marketplace) - AI CEO uses Compute for inference.

**Integration Points:**

1. **AI CEO Inference:**
   - AI CEO uses Compute marketplace for LLM inference
   - Compute providers get deep funding credit
   - Governance can decide Compute pricing/parameters

2. **Agent Inference:**
   - Governance agents can use Compute for AI tasks
   - Agents pay for inference via Compute marketplace

3. **Compute Governance:**
   - Governance can decide Compute marketplace parameters
   - Opinions can be about Compute configurations

**Implementation:**

```typescript
import { ComputeMarketplace } from '@jeju/compute';

class ComputeGovernanceIntegration {
  private compute: ComputeMarketplace;
  private aiCEO: AICEOService;
  private deepFunding: DeepFunding;
  
  async synthesizeOpinionsWithCompute(
    opinions: Opinion[]
  ): Promise<SynthesisResponse> {
    // AI CEO uses Compute marketplace for inference
    const inferenceRequest = {
      model: 'gpt-4',
      prompt: this.buildSynthesisPrompt(opinions),
      maxTokens: 4000
    };
    
    // Request inference from Compute marketplace
    const inferenceResult = await compute.requestInference(inferenceRequest);
    
    // Process inference result
    const synthesis = this.parseSynthesisResponse(inferenceResult);
    
    // Register Compute provider for deep funding credit
    await deepFunding.registerContribution(
      inferenceResult.provider,
      'AI inference for opinion synthesis',
      this.calculateInferenceCredit(inferenceResult),
      [],
      ContributionType.Other
    );
    
    return synthesis;
  }
  
  async createComputeOpinion(
    text: string,
    computeParameter: ComputeParameter
  ): Promise<number> {
    // Create app-scoped opinion for Compute
    const opinionId = await opinionStaking.createOpinion(
      text,
      ipfsHash,
      OpinionScope.App,
      compute.address,
      [computeToken, jejuToken] // Accept Compute and Jeju tokens
    );
    
    return opinionId;
  }
  
  async executeComputeProposal(
    proposalId: number,
    newParameters: ComputeParameters
  ): Promise<void> {
    // Execute Compute parameter change
    await compute.updateParameters(newParameters);
  }
  
  async getComputeState(): Promise<ComputeState> {
    return {
      providers: await compute.getProviders(),
      pricing: await compute.getPricing(),
      models: await compute.getAvailableModels()
    };
  }
}
```

---

## 5. DEPLOYMENT & OPERATIONS

### 5.1 Deployment Steps

1. **Deploy Smart Contracts:**
   ```bash
   npx hardhat deploy --network jeju
   ```

2. **Deploy AI Services:**
   ```bash
   docker-compose up -d
   ```

3. **Initialize Agent Registry:**
   ```typescript
   await agentRegistry.registerAgent(aiCEOAddress, "ai-ceo", "AI CEO for governance");
   await agentRegistry.registerAgent(technicalAgentAddress, "technical", "Technical implementation agent");
   // Specializations are free-form, market-driven
   ```

4. **Set Authorized Agents:**
   ```typescript
   await proposalExecution.authorizeAgent(aiCEOAddress);
   ```

### 5.2 Monitoring

- **On-Chain Events:** Monitor all contract events
- **AI Performance:** Track proposal quality and execution success
- **Opinion Trends:** Analyze opinion clustering and weight distribution
- **Deep Funding:** Monitor distribution accuracy and contributor satisfaction

---

## 6. SECURITY CONSIDERATIONS

### 6.1 Smart Contract Security

- **Reentrancy Guards:** All external calls protected
- **Access Control:** Only authorized agents can create proposals
- **Veto Mechanism:** High-stakes proposals require community veto
- **Time Locks:** Cooldown periods prevent gaming

### 6.2 AI Security

- **Multiple AIs:** Don't rely on single AI model
- **Human Oversight:** High-stakes decisions require human review
- **Transparency:** All AI reasoning is public
- **Performance Tracking:** Monitor AI performance and bias

### 6.3 Economic Security

- **Incentive Smoothing:** Diffusion + confusion prevent manipulation
- **Reputation Weighting:** Active participants weighted higher
- **Time Decay:** Recent opinions weighted higher
- **Veto Threshold:** Prevents small group manipulation

---

## 7. TESTING

### 7.1 Unit Tests

```typescript
describe('OpinionStaking', () => {
  it('should allow staking on opinions', async () => {
    const opinionId = await opinionStaking.createOpinion("Test opinion");
    await governanceToken.approve(opinionStaking.address, 1000);
    await opinionStaking.stakeOnOpinion(opinionId, 1000);
    
    const weight = await opinionStaking.getOpinionWeight(opinionId);
    expect(weight).to.be.gt(0);
  });
});
```

### 7.2 Integration Tests

```typescript
describe('Governance Flow', () => {
  it('should complete full governance cycle', async () => {
    // 1. Create opinion
    const opinionId = await createAndStakeOpinion("Improve block time");
    
    // 2. AI CEO synthesizes
    const synthesis = await aiCEO.synthesizeOpinions([opinionId]);
    
    // 3. Specialized agent creates proposal
    const proposal = await technicalAgent.generateProposal(synthesis);
    
    // 4. Execute proposal
    await proposalExecution.executeProposal(proposal.id);
    
    // 5. Verify execution
    const executed = await proposalExecution.proposals(proposal.id);
    expect(executed.executed).to.be.true;
  });
});
```

---

## 8. NEXT STEPS

1. Smart Contract Development: Implement core contracts
2. AI Service Development: Build AI CEO and agent competition system
3. Frontend Development: Build opinion staking UI and decision feed
4. Testing: Comprehensive test suite
5. Security Audit: External security audit
6. Deployment: Deploy to Jeju testnet
7. Beta Testing: Test with community
8. Mainnet Launch: Deploy to Jeju mainnet

---

**Document Status:** Technical Specification  
**Implementation:** Ready for development
