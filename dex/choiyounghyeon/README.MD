# 최영현

## LP Token mint imbalance(Dex.sol:addLiquidity:54)

**High (공격 파급력: Critical, 공격 난이도: Low)**

#### code

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint LPTokenAmount){

        require(tokenXAmount > 0 && tokenYAmount > 0);
        (uint256 reserveX, ) = amount_update();
        (, uint256 reserveY) = amount_update();

        if(totalSupply() == 0){ LPTokenAmount = tokenXAmount * tokenYAmount / 10**18;}
        **else{ LPTokenAmount = totalSupply() * tokenXAmount / reserveX;}**

        require(minimumLPTokenAmount <= LPTokenAmount);

        X.transferFrom(msg.sender, address(this), tokenXAmount);
        amountX = reserveX + tokenXAmount;
        Y.transferFrom(msg.sender, address(this), tokenYAmount);
        amountY = reserveY + tokenYAmount;

        _mint(msg.sender, LPTokenAmount);
    }
```

첫번째 LP 토큰 생성 이후 imbalnace 체크를 하지 않기 때문에 tokenXAmount를 많이 넣고, tokenYAmount를 적게 넣어서 많은 양의 LP 토큰을 mint할 수 있다.

#### PoC

```solidity
function testLPTokenAmount() external {
        uint lp1 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        uint lp2 = dex.addLiquidity(1000 ether, 1 ether, 0);
        assertTrue(lp1 == lp2,"");

        vm.startPrank(user1);
        dex.addLiquidity(100000 ether, 100000 ether, 0);
        vm.stopPrank();

        (uint rx, uint ry) = dex.removeLiquidity(lp2, 0, 0);
        console.log("INPUT X: %d", 1000 ether);
        console.log("INPUT Y: %d", 1 ether);
        console.log("REAL RX: %d", rx);
        console.log("REAL RY: %d", ry);
    }
```

```solidity
[PASS] testLPTokenAmount() (gas: 271911)
Logs:
  INPUT X: 1000000000000000000000
  INPUT Y: 1000000000000000000
  REAL RX: 1000000000000000000000
  REAL RY: 990205882352941176470
```

위와 같이 1000 ether, 1000 ether로 유동성을 공급했을 때의 LP token의 양과 1000 ether, 1 ether로 유동성을 공급했을 때 받아오는 LP token의 양이 동일하다.

removeLiquidity로 유동성을 빼내올때 받아오는 REAL RY의 양이 공급한 RY의 양보다 훨씬 많은것을 확인할 수 있다.

#### 해결방안

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint LPTokenAmount){

        require(tokenXAmount > 0 && tokenYAmount > 0);
        (uint256 reserveX, ) = amount_update();
        (, uint256 reserveY) = amount_update();

        if(totalSupply() == 0){ LPTokenAmount = tokenXAmount * tokenYAmount / 10**18;}
        else{
            uint liquidity_x = totalSupply() * tokenXAmount / reserveX;
            uint liquidity_y = totalSupply() * tokenYAmount / reserveY;
            **LPTokenAmount = (liquidity_x > liquidity_y) ? liquidity_y:liquidity_x;
            
            if(LPTokenAmount == liquidity_y){
                tokenXAmount = reserveX * tokenYAmount / reserveY;
            }else{
                tokenYAmount = reserveY * tokenXAmount / reserveX;
            }**
        }
        

        require(minimumLPTokenAmount <= LPTokenAmount);

        X.transferFrom(msg.sender, address(this), tokenXAmount);
        amountX = reserveX + tokenXAmount;
        Y.transferFrom(msg.sender, address(this), tokenYAmount);
        amountY = reserveY + tokenYAmount;

        _mint(msg.sender, LPTokenAmount);
    }
```

tokenXAmount, tokenYAmount가 imbalance하다는 점이 문제였기 때문에 LPTokenAmount를 X를 기준으로 하는게 아닌 X, Y둘중 작은 값이 나오는 토큰을 선택해서 해당 토큰을 기준으로 amount를 다시 설정한다.

```solidity
[FAIL. Reason: Assertion failed.] testLPTokenAmount() (gas: 283331)
Logs:
  Error: 
  Error: Assertion Failed
  INPUT X: 1000000000000000000000
  INPUT Y: 1000000000000000000
  REAL RX: 1000000000000000000
  REAL RY: 1000000000000000000
```

수정후 다시 test code를 돌려보면 위와 같이 의도한 수량만큼의 lp token과 rx, ry가 리턴됨을 확인할수 있다.

## Swap Large Amount (Dex.sol:swap:25-44)

******************medium (공격 파급력: Low, 공격 난이도: Medium)******************

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount) {
        require((tokenXAmount==0 && tokenYAmount>0) || (tokenYAmount==0 && tokenXAmount>0));
        require(amountX >0 && amountY > 0);

        (uint256 reserveX, uint256 reserveY) = amount_update();
        uint256 k = reserveX * reserveY;

        if(tokenXAmount > 0){
            **uint256 x_value = tokenXAmount / 1000 * 999;
            amountY = k / (amountX + x_value);**
            outputAmount = reserveY - amountY;

            require(outputAmount < amountY, "amountY is less than outputAmount");
            require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
            X.transferFrom(msg.sender, address(this), tokenXAmount);
            Y.transfer(msg.sender, outputAmount);
        }
        else{
            uint256 y_value = tokenYAmount / 1000 * 999;
            amountX = k / (amountY + y_value);
            outputAmount = reserveX - amountX;

            require(outputAmount < amountX, "amountX is less than outputAmount");
            require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
            Y.transferFrom(msg.sender, address(this), tokenYAmount);
            X.transfer(msg.sender, outputAmount);
        }
    }
```

Y'=(X*Y)/(X+X_*0.999)

결과적으로 bold처리 해놓은 부분을 수식으로 표현하자면 위와 같다

(X_: tokenXAmount, X: amountX, Y:amountY, Y’: Y-outputAmount)

Y’의 크기가 커질수록 outputAmount는 작아지게 되는데, 위 수식에서는 수수료를 분자에 들어간 tokenXAmount에 부과하고 있기 때문에 Y’이 커져서 outputAmount가 기대 수량의 한참 못미치게 된다.

#### PoC

```solidity
function testLargeSwap() external {
        uint lp = dex.addLiquidity(80000 ether, 80000 ether, 0);
        vm.startPrank(user1);
        uint output = dex.swap(70000 ether, 0, 0);
        console.log("70000: %d", 70000 ether);
        console.log("output: %d", output);
        vm.stopPrank();
    }
```

```solidity
[PASS] testLargeSwap() (gas: 209825)
Logs:
  70000: 70000000000000000000000
  output: 37313412926032148335891
```

#### 해결방안

해결 방안으로는 두가지가 있는것 같다.

1. outputAmount = (Y * (X_ * 0.999)) / (X + 0.999 * X_) ⇒ uniswap
2. outputAmount = Y - ((X * Y) / (X + X_))*0.999 ⇒ CPMM(X*Y=K)

## Swap Small Amount  Round Down(Dex.sol:swap:25-44)

**Informational (공격 파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount) {
        require((tokenXAmount==0 && tokenYAmount>0) || (tokenYAmount==0 && tokenXAmount>0));
        require(amountX >0 && amountY > 0);

        (uint256 reserveX, uint256 reserveY) = amount_update();
        uint256 k = reserveX * reserveY;

        if(tokenXAmount > 0){
            **uint256 x_value = tokenXAmount / 1000 * 999;
            amountY = k / (amountX + x_value);**
            outputAmount = reserveY - amountY;

            require(outputAmount < amountY, "amountY is less than outputAmount");
            require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
            X.transferFrom(msg.sender, address(this), tokenXAmount);
            Y.transfer(msg.sender, outputAmount);
        }
        else{
            uint256 y_value = tokenYAmount / 1000 * 999;
            amountX = k / (amountY + y_value);
            outputAmount = reserveX - amountX;

            require(outputAmount < amountX, "amountX is less than outputAmount");
            require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
            Y.transferFrom(msg.sender, address(this), tokenYAmount);
            X.transfer(msg.sender, outputAmount);
        }
    }
```

x_value를 계산할때 1000을 먼저 놔눠주고 있다. 만약 tokenXAmount가 999 wei라면 x_value=0이 되고, amountY = amountY가 되서 결국 outputAmount가 0이된다.

#### PoC

```solidity
function testSmallSwapRoundDown() external {
        uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
        uint output = dex.swap(999 wei, 0, 0);
        console.log("output: %d", output);
    }
```

```solidity
[FAIL. Reason: you claim too much token] testSmallSwapRoundDown() (gas: 184133)
Logs:
```

#### 해결방안

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount) {
        require((tokenXAmount==0 && tokenYAmount>0) || (tokenYAmount==0 && tokenXAmount>0));
        require(amountX >0 && amountY > 0);

        (uint256 reserveX, uint256 reserveY) = amount_update();
        uint256 k = reserveX * reserveY;

        if(tokenXAmount > 0){
            **uint256 x_value = (tokenXAmount * 10 ** 18) / 1000 * 999;
            amountY = (k * 10 ** 18) / ((amountX * 10 ** 18) + x_value);**
            outputAmount = reserveY - amountY;

            require(outputAmount < amountY, "amountY is less than outputAmount");
            require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
            X.transferFrom(msg.sender, address(this), tokenXAmount);
            Y.transfer(msg.sender, outputAmount);
        }
        else{
            uint256 y_value = tokenYAmount / 1000 * 999;
            amountX = k / (amountY + y_value);
            outputAmount = reserveX - amountX;

            require(outputAmount < amountX, "amountX is less than outputAmount");
            require(tokenMinimumOutputAmount < outputAmount, "you claim too much token");
            Y.transferFrom(msg.sender, address(this), tokenYAmount);
            X.transfer(msg.sender, outputAmount);
        }
    }
```

round down을 없애기 위해서 분자, 분모에 모두 10 ** 18만큼을 곱해서 round down을 완화해 줄 수 있다.

```solidity
[PASS] testSmallSwapRoundDown() (gas: 193102)
Logs:
  output: 999
```

다시 테스트케이스를 실행해보면 예상하고 있던 값이 나옴을 알 수 있다.
