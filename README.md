// SPDX-License-Identifier: MIT
// SmartWallet.sol — Custom smart wallet for FlashLoanBot-style trading
// --------------------------------------------------------------------------------
// WHAT THIS IS
//  A custom wallet contract, NOT an EOA wallet app. You deploy it from your
//  Trust Wallet on Sepolia, and your Trust Wallet becomes the owner.
//  The wallet contract itself holds any profit. The trading bot can only
//  trigger trades — it cannot take funds out.
//
//  It is FlashLoanBot, with full owner control added:
//   - owner: full control. Withdraw, pause, set / revoke tradingBot,
//     allow / block routers and tokens, set max loan.
//   - tradingBot: limited. Can only call requestFlashLoan(), and only
//     when NOT paused, only through an allowed router, only for an
//     allowed borrowed token, and only up to maxLoan.
//   - paused: owner can pause trading instantly. A paused wallet
//     rejects every requestFlashLoan(), even from the owner.
//     Withdrawals are NOT paused — the owner can always pull funds out.
//
// WHAT THIS IS NOT
//  - It is NOT a Safe / ERC-4337 account-abstraction wallet. For a
//    battle-tested smart wallet, use Safe on Sepolia instead of a
//    custom contract. This file is a learning / test template.
//  - It is UNAUDITED. Sepolia test funds only. Do NOT put real funds
//    in it, and do NOT deploy it on mainnet without a full review.
//  - It does NOT guarantee profit, and no "AI" can. See watcher notes:
//    a flash loan must be repaid in the same transaction, gas and
//    slippage eat profit, and faster bots compete for the same gap.
//    The watcher only trades when the simulated NET profit (after
//    premium and an estimated gas cost) clears your minimum.
//
// HOW TO DEPLOY (Sepolia only, from Trust Wallet, via Remix WalletConnect)
//  - Network: Sepolia, Chain ID 11155111.
//  - Constructor argument: Aave V3 PoolAddressesProvider, Sepolia:
//      0x012bAC54348C0E635dCAc9D5FB99f06F24136C9A
//    Verify it on Sepolia Etherscan before deploying. The Pool is
//    resolved from it at deploy time (expected Sepolia Pool:
//    0x6Ae43d3271ff6888e7Fc43Fd7321a503ff738951).
//  - After deploy, from the owner wallet, call:
//      setAllowedRouter(<router>, true)
//      setAllowedToken(<borrowed token>, true)
//      setTradingBot(<dedicated bot wallet>)  — only when you have one
//    Until a router and token are allowed, every trade is blocked.
//    Until tradingBot is set, only the owner can trade.

pragma solidity ^0.8.20;

interface IERC20 {
    function balanceOf(address account) external view returns (uint256);
    function approve(address spender, uint256 amount) external returns (bool);
    function transfer(address to, uint256 amount) external returns (bool);
}

interface IPoolAddressesProvider {
    function getPool() external view returns (address);
}

interface IPool {
    function flashLoanSimple(
        address receiverAddress,
        address asset,
        uint256 amount,
        bytes calldata params,
        uint16 referralCode
    ) external;
}

interface IFlashLoanSimpleReceiver {
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external returns (bool);
}

interface IRouter {
    function swapExactTokensForTokens(
        uint256 amountIn,
        uint256 amountOutMin,
        address[] calldata path,
        address to,
        uint256 deadline
    ) external returns (uint256[] memory amounts);
}

contract SmartWallet is IFlashLoanSimpleReceiver {
    address public immutable POOL;
    address public owner;
    address public tradingBot;
    bool public paused;
    uint256 public maxLoan;

    mapping(address => bool) public allowedRouters;
    mapping(address => bool) public allowedTokens;

    event FlashLoanRequested(address indexed asset, uint256 amount);
    event TradeExecuted(
        address indexed asset,
        uint256 amount,
        uint256 premium,
        uint256 amountOwed,
        uint256 profit
    );
    event Withdrawn(address indexed token, address indexed to, uint256 amount);
    event TradingBotSet(address indexed oldBot, address indexed newBot);
    event Paused(bool paused);
    event RouterAllowed(address indexed router, bool allowed);
    event TokenAllowed(address indexed token, bool allowed);
    event MaxLoanSet(uint256 maxLoan);

    error NotOwner();
    error NotAuthorized();
    error NotPool();
    error NotInitiator();
    error ZeroAmount();
    error PausedWallet();
    error RouterNotAllowed();
    error TokenNotAllowed();
    error LoanTooLarge();
    error InsufficientToRepay();

    modifier onlyOwner() {
        if (msg.sender != owner) revert NotOwner();
        _;
    }

    modifier onlyOwnerOrTradingBot() {
        if (msg.sender != owner && msg.sender != tradingBot)
            revert NotAuthorized();
        _;
    }

    struct TradeParams {
        address router;
        address[] pathOut;
        address[] pathBack;
        uint256 minOutFirstLeg;
        uint256 minOutSecondLeg;
    }

    constructor(address poolAddressesProvider, uint256 _maxLoan) {
        owner = msg.sender;
        maxLoan = _maxLoan;
        POOL = IPoolAddressesProvider(poolAddressesProvider).getPool();
    }

    // --- Full owner control ---

    function setTradingBot(address newBot) external onlyOwner {
        address oldBot = tradingBot;
        tradingBot = newBot;
        emit TradingBotSet(oldBot, newBot);
    }

    function setPaused(bool _paused) external onlyOwner {
        paused = _paused;
        emit Paused(_paused);
    }

    function setAllowedRouter(address router, bool allowed) external onlyOwner {
        allowedRouters[router] = allowed;
        emit RouterAllowed(router, allowed);
    }

    function setAllowedToken(address token, bool allowed) external onlyOwner {
        allowedTokens[token] = allowed;
        emit TokenAllowed(token, allowed);
    }

    function setMaxLoan(uint256 _maxLoan) external onlyOwner {
        maxLoan = _maxLoan;
        emit MaxLoanSet(_maxLoan);
    }

    // Owner can always withdraw, even when paused. Bot cannot.
    function withdrawToken(address token, address to, uint256 amount)
        external
        onlyOwner
    {
        IERC20(token).transfer(to, amount);
        emit Withdrawn(token, to, amount);
    }

    function withdrawETH(address payable to, uint256 amount)
        external
        onlyOwner
    {
        to.transfer(amount);
        emit Withdrawn(address(0), to, amount);
    }

    // --- Trading: owner or tradingBot, with all limits enforced ---

    function requestFlashLoan(
        address asset,
        uint256 amount,
        address router,
        address[] calldata pathOut,
        address[] calldata pathBack,
        uint256 minOutFirstLeg,
        uint256 minOutSecondLeg
    ) external onlyOwnerOrTradingBot {
        if (paused) revert PausedWallet();
        if (amount == 0) revert ZeroAmount();
        if (!allowedRouters[router]) revert RouterNotAllowed();
        if (!allowedTokens[asset]) revert TokenNotAllowed();
        if (amount > maxLoan) revert LoanTooLarge();

        TradeParams memory trade = TradeParams({
            router: router,
            pathOut: pathOut,
            pathBack: pathBack,
            minOutFirstLeg: minOutFirstLeg,
            minOutSecondLeg: minOutSecondLeg
        });

        emit FlashLoanRequested(asset, amount);

        IPool(POOL).flashLoanSimple(
            address(this),
            asset,
            amount,
            abi.encode(trade),
            0
        );
    }

    // Called by Aave only, in the same transaction as the loan.
    function executeOperation(
        address asset,
        uint256 amount,
        uint256 premium,
        address initiator,
        bytes calldata params
    ) external override returns (bool) {
        if (msg.sender != POOL) revert NotPool();
        if (initiator != address(this)) revert NotInitiator();

        TradeParams memory trade = abi.decode(params, (TradeParams));
        uint256 amountOwed = amount + premium;

        IERC20(asset).approve(trade.router, amount);
        IRouter(trade.router).swapExactTokensForTokens(
            amount,
            trade.minOutFirstLeg,
            trade.pathOut,
            address(this),
            block.timestamp
        );

        address middleToken = trade.pathOut[trade.pathOut.length - 1];
        uint256 middleBalance = IERC20(middleToken).balanceOf(address(this));
        IERC20(middleToken).approve(trade.router, middleBalance);
        IRouter(trade.router).swapExactTokensForTokens(
            middleBalance,
            trade.minOutSecondLeg,
            trade.pathBack,
            address(this),
            block.timestamp
        );

        uint256 finalBalance = IERC20(asset).balanceOf(address(this));
        if (finalBalance < amountOwed) revert InsufficientToRepay();

        IERC20(asset).approve(POOL, amountOwed);

        emit TradeExecuted(
            asset,
            amount,
            premium,
            amountOwed,
            finalBalance - amountOwed
        );
        return true;
    }

    receive() external payable {}
}
# Tech-World