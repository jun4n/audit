# 구민재

## public transfer & mint LP token
#### Dex.sol:transfer:105
****critical (공격 파급력: High, 공격 난이도: Low)****

```solidity
function transfer(address to, uint256 lpAmount) public override(ERC20, IDex) returns (bool) {
    _mint(to, lpAmount);
    return true;
}
```

ERC20을 상속받은 컨트랙트에서 transfer함수를 오버라이드 해서 _mint함수를 호출하게 바꿨다.

transfer의 가시성 지정자가 public이고, 다른 인증요소가 없기 때문에 공격자는 transfer함수를 호출해서 원하는대로 LP token을 민팅할 수 있다.

#### PoC

```solidity
function testTransferVisibility() external {
    vm.startPrank(user1);
    dex.transfer(address(this), 1000 ether);
    uint amount = dex.balanceOf(address(this));
    console.log("%d", amount);
    amount = dex.totalSupply();
    console.log("%d", amount);
    vm.stopPrank();
    assertTrue(!(amount == 1000 ether), "testTransferVisibility Error");
}
```

```solidity
[FAIL. Reason: Assertion failed.] testTransferVisibility() (gas: 73929)
Logs:
  1000000000000000000000
  1000000000000000000000
  Error: testTransferVisibility Error
  Error: Assertion Failed
```

#### 해결방안

문제의 원인은 transfer함수를 오버라이딩 해서 _mint함수를 호출하고, 가시성 지정자가 public인 상황에 있기 때문에 transfer함수를 오버라이딩 하지 않고, _mint함수를 이용해서 민팅을 해야한다.

## addLiquidity Round down
#### Dex.sol:addLiquidity:31
************************************low (파급력: medium, 공격 난이도: low)************************************

```
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPToeknAmount) public returns (uint256 LPTokenAmount) {
    require(tokenXAmount > 0 && tokenYAmount > 0, "Token must be not zero.");
    require(balanceOf(msg.sender) >= LPTokenAmount, "Insufficient LP.");
    reserveX = tokenX.balanceOf(address(this));
    reserveY = tokenY.balanceOf(address(this));
    uint256 liquidity;

    uint256 _totalSupply = totalSupply();
    if (_totalSupply == 0) {
        LPTokenAmount = _sqrt((tokenXAmount + reserveX) * (tokenYAmount + reserveY) / MINIMUM_LIQUIDITY);
    } else {
        require(reserveX * tokenYAmount == reserveY * tokenXAmount, "Add Liquidity Error");
        LPTokenAmount = _min(_totalSupply * tokenXAmount / reserveX, _totalSupply * tokenYAmount / reserveY);
    }
    require(LPTokenAmount >= minimumLPToeknAmount, "Minimum LP Error");
    _mint(msg.sender, LPTokenAmount);
    reserveX += tokenXAmount;
    reserveY += tokenYAmount;
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);

    emit AddLiquidity(msg.sender, tokenXAmount, tokenYAmount);

}
```

LP token을 처음 발급할때 MINIMUM_LIQUDITY로 토큰들의 공급량의 곱을 나눈다음에 루트를 씌운다.

LP token의 첫 발급된 크기가 작기 때문에 그 이후의 토큰들도 작게 발급되는 상황이 발생한다.

토큰들의 단위가 작기 때문에 나눗셈을 진행할 때 Round Down이 생길 가능성도 커진다.

#### PoC

```solidity
function testAddLiquidity() external {
    uint lp1 = dex.addLiquidity(1000 ether, 1000 ether, 0);
    uint lp2 = dex.addLiquidity(1000 ether, 1000 ether, 0);
    uint lp3 = dex.addLiquidity(100 ether, 100 ether, 0);
    console.log("LP1: %d", lp1);
    console.log("LP2: %d", lp2);
    console.log("LP3: %d", lp3);

    (uint rx, uint ry) = dex.removeLiquidity(lp3, 0, 0);
    console.log("EX: %d", 100 ether);
    console.log("rx: %d", rx);
    console.log("EY: %d", 100 ether);
    console.log("ry: %d", ry);
}
```

```solidity
[PASS] testAddLiquidity() (gas: 256066)
Logs:
  LP1: 31622776601683793319
  LP2: 31622776601683793319
  LP3: 3162277660168379331
  EX: 100000000000000000000
  rx: 99999999999999999972
  EY: 100000000000000000000
  ry: 99999999999999999972
```

#### 해결방안

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPToeknAmount) public returns (uint256 LPTokenAmount) {
    require(tokenXAmount > 0 && tokenYAmount > 0, "Token must be not zero.");
    require(balanceOf(msg.sender) >= LPTokenAmount, "Insufficient LP.");
    reserveX = tokenX.balanceOf(address(this));
    reserveY = tokenY.balanceOf(address(this));
    uint256 liquidity;

    uint256 _totalSupply = totalSupply();
    if (_totalSupply == 0) {
        LPTokenAmount = _sqrt((tokenXAmount + reserveX) * (tokenYAmount + reserveY));
    } else {
        require(reserveX * tokenYAmount == reserveY * tokenXAmount, "Add Liquidity Error");
        LPTokenAmount = _min(_totalSupply * tokenXAmount / reserveX, _totalSupply * tokenYAmount / reserveY);
    }
    require(LPTokenAmount >= minimumLPToeknAmount, "Minimum LP Error");
    _mint(msg.sender, LPTokenAmount);
    reserveX += tokenXAmount;
    reserveY += tokenYAmount;
    tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    tokenY.transferFrom(msg.sender, address(this), tokenYAmount);

    emit AddLiquidity(msg.sender, tokenXAmount, tokenYAmount);

}
```

```solidity
[PASS] testAddLiquidity() (gas: 257258)
Logs:
  LP1: 1000000000000000000000
  LP2: 1000000000000000000000
  LP3: 100000000000000000000
  EX: 100000000000000000000
  rx: 100000000000000000000
  EY: 100000000000000000000
  ry: 100000000000000000000
```

첫 발급량이 10 **6만큼 나눠져서 들어가는게 문제이기 때문에 MINIMUM_LIQUIDITY로 나누지 않고 첫번째 토큰을 발급한다.
