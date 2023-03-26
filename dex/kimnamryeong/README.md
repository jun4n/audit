# 김남령

## minimumTokenAmount 등호 생략

****************Informational (공격 파급력: Low, 공격 난이도: Low)****************

```solidity
function removeLiquidity(uint256 _LPTokenAmount, uint256 _minimumTokenXAmount, uint256 _minimumTokenYAmount) external returns(uint256, uint256){
    require(_LPTokenAmount > 0, "INSUFFICIENT_AMOUNT");
    require(balanceOf(msg.sender) >= _LPTokenAmount, "INSUFFICIENT_LPtoken_AMOUNT");
    
    uint256 reserveX;
    uint256 reserveY;
    uint256 amountX;
    uint256 amountY;

    (reserveX, reserveY) = _update();

    amountX =  reserveX * _LPTokenAmount/ totalSupply();
    amountY = reserveY * _LPTokenAmount / totalSupply();
    console.log("%d, %d", amountX, _minimumTokenXAmount);
    require(amountX > _minimumTokenXAmount && amountY > _minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
    console.logBool(amountX > _minimumTokenXAmount);
    tokenX_.transfer(msg.sender, amountX);
    tokenY_.transfer(msg.sender, amountY);

    _burn(msg.sender, _LPTokenAmount);

    (reserveX_,reserveY_) = _update();
    return (amountX, amountY);
}
```

require()를 통해서 사용자가 입력한 최소한의 X, Y amount를 맞춰주는 조건이 있는데, 사용자가 정확히 받을 수 있는 토큰의 개수를 전달할 경우 revert가 난다.

#### PoC

```solidity
function testExactRemoveLiquidity() external {
    uint firstLPReturn = dex.addLiquidity(1000 ether, 1000 ether, 0);
    emit log_named_uint("firstLPReturn", firstLPReturn);

    uint secondLPReturn = dex.addLiquidity(1000 ether * 2, 1000 ether * 2, 0);
    emit log_named_uint("secondLPReturn", secondLPReturn);

    (uint rx, uint ry) = dex.removeLiquidity(secondLPReturn, 2000 ether, 2000 ether);
    console.log("%d %d", rx, ry);
    assertTrue(rx == 2000 ether);
    assertTrue(ry == 2000 ether);
}
```

```solidity
[FAIL. Reason: INSUFFICIENT_LIQUIDITY_BURNED] testExactRemoveLiquidity() (gas: 205180)
Logs:
  firstLPReturn: 1000000000000000000000
  secondLPReturn: 2000000000000000000000
```

정확히 2000 ether를 유동성을 빼면서 받을 수 있고, 해당 값을 전달해서 revert가 나왔다.

#### 해결방안

```solidity
require(amountX >= _minimumTokenXAmount && amountY >= _minimumTokenYAmount, "INSUFFICIENT_LIQUIDITY_BURNED");
```

## **Swap Small Amount Round Down**

**Informational (공격 파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 _tokenXAmount, uint256 _tokenYAmount, uint256 _tokenMinimumOutputAmount) external returns (uint256){
    require((_tokenXAmount==0) && (_tokenYAmount >0) || (_tokenYAmount==0)&&(_tokenXAmount>0),"INSUFFICIENT_AMOUNT");
    
    require(reserveX_ >0 && reserveY_ >0, "fail");

    uint256 reserveX;
    uint256 reserveY;
    (reserveX, reserveY) = _update();
    
    uint256 inputWithFees;
    uint256 outputAmount;

    if(_tokenXAmount > 0){
        inputWithFees = _tokenXAmount * 999 / 1000;
        outputAmount =  (reserveY * inputWithFees)/(reserveX + inputWithFees);
        require(outputAmount >= _tokenMinimumOutputAmount, "INSUFICIENT_OUTPUT_AMOUNT");
        
        tokenX_.transferFrom(msg.sender, address(this), _tokenXAmount);
        tokenY_.transfer(msg.sender, outputAmount);
    }
    else{
        inputWithFees = _tokenYAmount * 999 /1000;
        outputAmount = (reserveX * inputWithFees) / (reserveY + inputWithFees);
        require(outputAmount >= _tokenMinimumOutputAmount, "INSUFICIENT_OUTPUT_AMOUNT");

        tokenY_.transferFrom(msg.sender, address(this), _tokenYAmount);
        tokenX_.transfer(msg.sender, outputAmount);
    }

    (reserveX_,reserveY_) = _update();
    return outputAmount;

}
```

다른 사람들과 동일하게 수수료를 제외한 값을 tokenAmountX에 곱해주는 과정에서 roundDown이 일어난다.

```solidity
function testSmallSwapRoundDown() external {
    uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
    uint output = dex.swap(999 wei, 0, 0);
    console.log("output: %d", output);
}
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 193996)
Logs:
  output: 997
```

#### 해결방안

```solidity
function swap(uint256 _tokenXAmount, uint256 _tokenYAmount, uint256 _tokenMinimumOutputAmount) external returns (uint256){
    require((_tokenXAmount==0) && (_tokenYAmount >0) || (_tokenYAmount==0)&&(_tokenXAmount>0),"INSUFFICIENT_AMOUNT");
    
    require(reserveX_ >0 && reserveY_ >0, "fail");

    uint256 reserveX;
    uint256 reserveY;
    (reserveX, reserveY) = _update();
    
    uint256 inputWithFees;
    uint256 outputAmount;

    if(_tokenXAmount > 0){
        inputWithFees = (_tokenXAmount * 10 ** 18) * 999 / 1000;
        outputAmount =  (reserveY * inputWithFees)/(reserveX* 10 **18 + inputWithFees);
        require(outputAmount >= _tokenMinimumOutputAmount, "INSUFICIENT_OUTPUT_AMOUNT");
        
        tokenX_.transferFrom(msg.sender, address(this), _tokenXAmount);
        tokenY_.transfer(msg.sender, outputAmount);
    }
    else{
        inputWithFees = _tokenYAmount * 999 /1000;
        outputAmount = (reserveX * inputWithFees) / (reserveY + inputWithFees);
        require(outputAmount >= _tokenMinimumOutputAmount, "INSUFICIENT_OUTPUT_AMOUNT");

        tokenY_.transferFrom(msg.sender, address(this), _tokenYAmount);
        tokenX_.transfer(msg.sender, outputAmount);
    }

    (reserveX_,reserveY_) = _update();
    return outputAmount;

}
```

ouputAmount의 분자와 분모에 10 ** 18을 곱해주는데, 여기서는 inputWithFees가 결과적으로 분자와 분모에 들어가기 때문에 inputWithFees값과, reserveX에 곱해주었다.
