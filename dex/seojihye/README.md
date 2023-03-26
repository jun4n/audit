# 서지혜

## swap small amount round down

**Informational (파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) public returns (uint256){
    require(((tokenXAmount == 0) && (tokenYAmount > 0)) || ((tokenXAmount > 0) && (tokenYAmount == 0)), "only one amount should be zero");
    require(_amountX > 0 && _amountY > 0, "no token to swap");

    updateTokenBalance();
    
    uint256 amount_;
    if(tokenXAmount > 0){
        amount_ = _amountY * (tokenXAmount * 999 / 1000) / (_amountX + (tokenXAmount * 999 / 1000));
        
        require(amount_ >= tokenMinimumOutputAmount, "less than minimum swap amount");
        _amountY -= amount_ ;
        _amountX += tokenXAmount;
        _tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
        _tokenY.transfer(msg.sender, amount_ );
    }
    else{
        amount_ = _amountX * (tokenYAmount * 999 / 1000) / (_amountY + (tokenYAmount * 999 / 1000));

        require(amount_ >= tokenMinimumOutputAmount, "less than minimum swap amount");
        _amountX -= amount_;
        _amountY += tokenXAmount;
        _tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
        _tokenX.transfer(msg.sender, amount_);
    }
    return amount_;
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
[PASS] testSmallSwapRoundDown() (gas: 197008)
Logs:
  output: 997
```

#### 해결방안

amount_를 구할때 분자와 분모에 10 ** 18을 곱해서 단위를 올려서 round down을 완화시킨다.

## addLiquidity Imbalance

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) public returns (uint256){
    require(tokenXAmount > 0, "token x amount is 0");
    require(tokenYAmount > 0, "token y amount is 0");
    require(_tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
    require(_tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
    require(_tokenX.balanceOf(msg.sender) >= tokenXAmount, "ERC20: transfer amount exceeds balance");
    require(_tokenY.balanceOf(msg.sender) >= tokenYAmount, "ERC20: transfer amount exceeds balance");
    
    updateTokenBalance();

    // what if _amountX/_amountY and tokenXAmount/tokenYAmount is different?
    uint lpAmount;
    if(totalSupply() == 0){
        lpAmount = tokenXAmount * tokenYAmount / _decimal; // is amount best? no overflow?
    }
    else{
        require(_decimal * tokenXAmount / tokenYAmount == _decimal * _amountX / _amountY, "amount breaks the pool ratio");
        lpAmount = totalSupply() * tokenXAmount / _amountX;
    }
    require(lpAmount >= minimumLPTokenAmount, "less than minimum lp token amount");
    _amountX += tokenXAmount;
    _amountY += tokenYAmount;
    _tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    _tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
    _mint(msg.sender, lpAmount);
    retu
```

imbalance를 체크하기 위해서 require구 문으로 체크하고 있다.

하지만 만약 첫번째 addBalance에서 1wei, 10 ehter를 넣고, 두번째 addBalance에서 99wei, 100 ether를 넣게 되면 0==0이되기 때문에 imbalance check를 깰 수 있다.

#### PoC

```solidity
function testAddLiquidityImbalanceInput() external {
    uint lp = dex.addLiquidity(1 wei, 10 ether, 0);
    vm.startPrank(user1);
    uint lp2 = dex.addLiquidity(99 wei, 100 ether, 0);
    vm.stopPrank();
    console.log("lp: %d", lp);
    console.log("lp2: %d", lp2);
}
```

```solidity
[PASS] testAddLiquidityImbalanceInput() (gas: 246656)
Logs:
  lp: 10
  lp2: 990
```

#### 해결방안

 tokenXAmount  * amountY ==  _amountX * tokenYAmount를 통해서 imbalance 체크를 한다.
