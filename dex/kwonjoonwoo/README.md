# 권준우

## swap small amount round down

**Informational (파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) public returns (uint256 outputAmount){
    require(tokenXAmount >= 0 || tokenYAmount >= 0);
    require(tokenXAmount == 0 || tokenYAmount == 0);
    uint256 xBalance = tokenX.balanceOf(address(this));
    uint256 yBalance = tokenY.balanceOf(address(this));
    
    if(tokenXAmount != 0){
        outputAmount = yBalance * (tokenXAmount * 999 / 1000) / (xBalance + (tokenXAmount * 999 / 1000));

        require(outputAmount >= tokenMinimumOutputAmount);
        yBalance -= outputAmount ;
        xBalance += tokenXAmount;
        tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
        tokenY.transfer(msg.sender, outputAmount );
    }
    else{
        outputAmount = xBalance * (tokenYAmount * 999 / 1000) / (yBalance + (tokenYAmount * 999 / 1000));

        require(outputAmount >= tokenMinimumOutputAmount);
        xBalance -= outputAmount;
        yBalance += tokenYAmount;
        tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
        tokenX.transfer(msg.sender, outputAmount);
    }
    return outputAmount;
}
```

uniswap v2에서 사용한 수수료 매기는 방식은 tokenXAmoutnt에 0.999를 곱해주는 방식이고, 이 때 round down이 발생할 수 있다.

#### PoC

```solidity
function testSmallSwapRoundDown() external {
        uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
        uint output = dex.swap(999 wei, 0, 0);
        console.log("output: %d", output);
    }
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 186125)
Logs:
  output: 997
```

#### 해결방안

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) public returns (uint256 outputAmount){
    require(tokenXAmount >= 0 || tokenYAmount >= 0);
    require(tokenXAmount == 0 || tokenYAmount == 0);
    uint256 xBalance = tokenX.balanceOf(address(this));
    uint256 yBalance = tokenY.balanceOf(address(this));
    
    if(tokenXAmount != 0){
        outputAmount = yBalance * ((tokenXAmount * 10 ** 18) * 999 / 1000) / ((xBalance * 10 ** 18) + ((tokenXAmount * 10 ** 18) * 999 / 1000));

        require(outputAmount >= tokenMinimumOutputAmount);
        yBalance -= outputAmount ;
        xBalance += tokenXAmount;
        tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
        tokenY.transfer(msg.sender, outputAmount );
    }
    else{
        outputAmount = xBalance * (tokenYAmount * 999 / 1000) / (yBalance + (tokenYAmount * 999 / 1000));

        require(outputAmount >= tokenMinimumOutputAmount);
        xBalance -= outputAmount;
        yBalance += tokenYAmount;
        tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
        tokenX.transfer(msg.sender, outputAmount);
    }
    return outputAmount;
}
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 186398)
Logs:
  output: 998
```

oupoutAmount를 계산할때 분자 분모에 10**18을 곱해줘서 Round down을 완화시켜준다.

## AddLiquidity imbalance input

**medium (파급력: medium, 공격 난이도: Medium)**

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
    require(tokenXAmount > 0 && tokenYAmount > 0,"please check tokenXAmount and tokenYAmount");
    uint256 xBalance = balances[address(tokenX)];
    uint256 yBalance = balances[address(tokenY)];
    uint256 liquidityX;
    uint256 liquidityY;
    if(totalLiquidity == 0) {
        LPTokenAmount = Math.sqrt(tokenXAmount * tokenYAmount);
        require(LPTokenAmount >= minimumLPTokenAmount);
        totalLiquidity = LPTokenAmount;
    } 
    else {
        liquidityX = (totalLiquidity*tokenXAmount) / xBalance;
        liquidityY = (totalLiquidity*tokenYAmount) / yBalance;
        LPTokenAmount = (liquidityX < liquidityY) ? liquidityX : liquidityY;
        require(LPTokenAmount >= minimumLPTokenAmount);
        totalLiquidity += LPTokenAmount;
    }
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
    balances[address(tokenX)] += tokenXAmount;
    balances[address(tokenY)] += tokenYAmount;
    LPToken_balances[msg.sender] += LPTokenAmount;
    return LPTokenAmount;
}
```

tokenX, tokenY가 imbalance하게 들어오더라도 모두 풀에 들어가게 되어 있기 때문에 풀의 균형을 깨고 이득을 보는 공격이 가능하다.

#### PoC

공격 시나리오는 다음과 같다.

1. A는 100 ETH를 가지고 있고, ETH price = 4000 ether, USDC price = 1 ether인 상황
2. Lending protocol의 ETH-USDC 풀에 ETH조금과 USDC를 엄청 많이 addLiquidity한다.
3. 풀의 토큰 비율이 변경되었으므로 A는 변경된 비율로 ETH를 USDC로 swap한다.
4. 기존의 ETH로 얻을 수 있는 USDC의 양보다 많은 양의 USDC를 swap할 수 있게 된다

```solidity
function testManyETH() external {
    dex.addLiquidity(4000000 ether, 1000 ether, 0);
    vm.startPrank(user1);
    uint lp = dex.addLiquidity(8000000 ether, 1 wei, 0);
    uint outputUSDC = dex.swap(0, 1000 ether, 0);
    console.log("outputUSDC: %d", outputUSDC);
    console.log("1000*4000: %d", 1000 * 4000 ether);
    vm.stopPrank();
}
```

```solidity
[PASS] testManyETH() (gas: 239914)
Logs:
  outputUSDC: 5996998499249624812403203
  1000*4000: 4000000000000000000000000
```

#### 해결방안

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
    require(tokenXAmount > 0 && tokenYAmount > 0,"please check tokenXAmount and tokenYAmount");
    uint256 xBalance = balances[address(tokenX)];
    uint256 yBalance = balances[address(tokenY)];
    uint256 liquidityX;
    uint256 liquidityY;
    if(totalLiquidity == 0) {
        LPTokenAmount = Math.sqrt(tokenXAmount * tokenYAmount);
        require(LPTokenAmount >= minimumLPTokenAmount);
        totalLiquidity = LPTokenAmount;
    } 
    else {
        liquidityX = (totalLiquidity*tokenXAmount) / xBalance;
        liquidityY = (totalLiquidity*tokenYAmount) / yBalance;
        LPTokenAmount = (liquidityX < liquidityY) ? liquidityX : liquidityY;
        require(LPTokenAmount >= minimumLPTokenAmount);
        totalLiquidity += LPTokenAmount;
        if(LPTokenAmount == liquidityX){
            tokenYAmount = tokenXAmount * yBalance / xBalance;
        }else{
            tokenXAmount = tokenYAmount * xBalance / yBalance;
        }
    }
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
    balances[address(tokenX)] += tokenXAmount;
    balances[address(tokenY)] += tokenYAmount;
    LPToken_balances[msg.sender] += LPTokenAmount;
    return LPTokenAmount;
}
```

```solidity
[PASS] testManyETH() (gas: 240096)
Logs:
  outputUSDC: 1998999499749874937469733
  1000*4000: 4000000000000000000000000
```

풀의 비율과 상관없이 제공된 모든 토큰을 풀에 집어넣은게 문제이기 때문에 해당 부분을 수정한다.
