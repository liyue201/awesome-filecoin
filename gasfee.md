
## 矿工费

```
minerFee = GasPremium  *  gasLimit
```

## Gas销毁

 gasLimit - 1.1 * gasUsed >= 0 时 :
```
gasBurn = ((gasLimit - gasUsed) * (gasLimit - 1.1 * gasUsed) / gasUsed + gasUsed) * baseFee
```

当 gasLimit - 1.1 * gasUsed < 0 时
```
gasBurn = gasUsed * baseFee
```

当gasLimit - 1.1 * gasUsed > gasUsed 时
```
gasBurn = gasLimit * baseFee
```

 这个函数的返回值是gasToBurn，  Gas销毁公式为（gasToBurn + gasUsed） * baseFee
 ```go
// ComputeGasOverestimationBurn computes amount of gas to be refunded and amount of gas to be burned
// Result is (refund, burn)
func ComputeGasOverestimationBurn(gasUsed, gasLimit int64) (int64, int64) {
	if gasUsed == 0 {
		return 0, gasLimit
	}

	// over = gasLimit/gasUsed - 1 - 0.1
	// over = min(over, 1)
	// gasToBurn = (gasLimit - gasUsed) * over

	// so to factor out division from `over`
	// over*gasUsed = min(gasLimit - (11*gasUsed)/10, gasUsed)
	// gasToBurn = ((gasLimit - gasUsed)*over*gasUsed) / gasUsed
	over := gasLimit - (gasOveruseNum*gasUsed)/gasOveruseDenom
	if over < 0 {
		return gasLimit - gasUsed, 0
	}

	// if we want sharper scaling it goes here:
	// over *= 2

	if over > gasUsed {
		over = gasUsed
	}

	// needs bigint, as it overflows in pathological case gasLimit > 2^32 gasUsed = gasLimit / 2
	gasToBurn := big.NewInt(gasLimit - gasUsed)
	gasToBurn = big.Mul(gasToBurn, big.NewInt(over))
	gasToBurn = big.Div(gasToBurn, big.NewInt(gasUsed))

	return gasLimit - gasUsed - gasToBurn.Int64(), gasToBurn.Int64()
}
```
