# Rubicon Finance

# Pre

源码地址：[https://github.com/sherlock-audit/2024-02-rubicon-finance/tree/main](https://github.com/sherlock-audit/2024-02-rubicon-finance/tree/main)

# Security Issues Review

## 1. 🟡 RubiconFeeController.sol &

### 1.1 由于费用四舍五入不一致，订单可能会revert

RubiconFeeController.sol::getFeeOutputs#L81：

```solidity
uint256 feeAmount = fee.applyFee
                ? order.outputs[i].amount.mulDivUp(fee.fee, DENOM)
                : order.outputs[i].amount.mulDivUp(baseFee, DENOM);
```

此处计算feeAmount会向上四舍五入。

在ProtocolFees.sol#L44中会调用此方法获取`feeOutputs` ，在检测阶段：

```solidity
if (feeOutput.amount > tokenValue.mulDivDown(MAX_FEE, DENOM)) {
                revert FeeTooLarge(
                    feeOutput.token,
                    feeOutput.amount,
                    feeOutput.recipient
                );
            }
```

进行判断时会进行向下的四舍五入，因此可能会触发revert。