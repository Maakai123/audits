## Stability DAO
[Contest details](https://cantina.xyz/competitions/e1c0be8d-0c3d-485a-a446-a582beb120b1)

### [Medium-01] Missing Slippage Control in WrappedMetaVault

**Description**

The WrappedMetaVault contract lacks slippage control in its deposit and withdraw functions, allowing users to receive fewer shares or assets than expected due to share price manipulation or front-running. This violates best practices for ERC4626 vaults, exposing users to financial losses and breaking expected security guarantees.

The WrappedMetaVault contract, which implements the ERC4626 standard, does not enforce minimum output checks (slippage control) for deposits and withdrawals. Slippage control ensures that users receive at least a specified minimum number of shares (minShares) during deposits or assets (minAssets) during withdrawals, protecting against share price changes due to front-running, market volatility, or malicious manipulation (e.g., share inflation attacks)..

Users expect to receive shares or assets proportional to the vault’s current share price, as estimated by previewDeposit or previewWithdraw. Without slippage control, users can receive significantly less due to external manipulation, violating fairness.

Protection Against Manipulation: The absence of slippage checks allows attackers to manipulate the vault’s share price (e.g., by donating assets to the underlying metaVault), causing financial losses for users.

Reliability: External contracts relying on predictable deposit/withdrawal outcomes may fail if the share price changes unexpectedly, breaking integration reliability.

function _deposit(address caller, address receiver, uint assets, uint shares) internal override {
    WrappedMetaVaultStorage storage $ = _getWrappedMetaVaultStorage();
    if ($.isMulti) {
        address _metaVault = $.metaVault;
        address[] memory _assets = new address[](1);
        _assets[0] = asset();
        uint[] memory amountsMax = new uint[](1);
        amountsMax[0] = assets;
        IERC20(_assets[0]).safeTransferFrom(caller, address(this), assets);
        _mint(receiver, shares); // No minShares check
        IERC20(_assets[0]).forceApprove(_metaVault, assets);
        IStabilityVault(_metaVault).depositAssets(_assets, amountsMax, 0, address(this));
        emit Deposit(caller, receiver, assets, shares);
    } else {
        super._deposit(caller, receiver, assets, shares);
    }
}
The function mints shares without checking if they meet a minimum threshold, leaving users vulnerable to receiving fewer shares if the metaVault’s share price is manipulated.

function _withdraw(address caller, address receiver, address owner, uint assets, uint shares) internal override {
    WrappedMetaVaultStorage storage $ = _getWrappedMetaVaultStorage();
    if ($.isMulti) {
        if (caller != owner) {
            _spendAllowance(owner, caller, shares);
        }
        address[] memory _assets = new address[](1);
        _assets[0] = asset();
        IStabilityVault($.metaVault).withdrawAssets(
            _assets,
            (assets + 1) * 10 ** (18 - IERC20Metadata(asset()).decimals()),
            new uint[](1),
            receiver,
            address(this)
        ); // No minAssets check
        _burn(owner, shares);
        emit Withdraw(caller, receiver, owner, assets, shares);
    } else {
        super._withdraw(caller, receiver, owner, assets, shares);
    }
}
The function withdraws assets without ensuring the actual assets received meet a minimum threshold, exposing users to losses if the share price drops.

## Impact Explanation
Financial Loss: Users can lose significant value due to receiving fewer shares or assets than expected. For example, in the PoC, a user depositing 100 tokens received ~90.91% fewer shares, equivalent to a 1,000-token loss at the inflated share price.

**Recommendation**

To fix the missing slippage control, modify the deposit and withdraw functions to include minShares and minAssets parameters, ensuring users receive at least the expected outputs.

