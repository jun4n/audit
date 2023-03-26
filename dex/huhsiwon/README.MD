# 허시원

## Swap Small Amount Round Down
#### Dex.sol:swap:87
**Informational (공격 파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) public returns (uint256 outputAmount){
        require(!(tokenXAmount == 0 && tokenYAmount == 0), "invalid input");
        require(!(tokenXAmount != 0 && tokenYAmount != 0), "invalid input");

        tokenX_in_LP = tokenX.balanceOf(address(this));
        tokenY_in_LP = tokenY.balanceOf(address(this));
        
        if(tokenXAmount != 0){
            **outputAmount = tokenY_in_LP * (tokenXAmount * 999 / 1000) / (tokenX_in_LP + (tokenXAmount * 999 / 1000));**

            require(outputAmount >= tokenMinimumOutputAmount, "minimum ouput amount check failed");
            tokenY_in_LP -= outputAmount ;
            tokenX_in_LP += tokenXAmount;
            tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
            tokenY.transfer(msg.sender, outputAmount );
        }
        else{
            outputAmount = tokenX_in_LP * (tokenYAmount * 999 / 1000) / (tokenY_in_LP + (tokenYAmount * 999 / 1000));

            require(outputAmount >= tokenMinimumOutputAmount, "minimum ouput amount check failed");
            tokenX_in_LP -= outputAmount;
            tokenY_in_LP += tokenXAmount;
            tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
            tokenX.transfer(msg.sender, outputAmount);
        }
        return outputAmount;
    }
```

outputAmount = 

tokenY_in_LP * (tokenXAmount * 999 / 1000) / (tokenX_in_LP + (tokenXAmount * 999 / 1000))

수수료를 이미 붙이고 outputAmount를 계산하기 때문에 round down이 발생하기 쉽다.

#### PoC

```solidity
function testSmallSwapRoundDown() external {
        uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
        uint output = dex.swap(999 wei, 0, 0);
        console.log("output: %d", output);
    }
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 201004)
Logs:
  output: 997
```

최영현님이랑 다른점은 999를 먼저 곱해줬기 때문에 0이 나오는 불상사는 일어나지 않았따.

#### 해결방안

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) public returns (uint256 outputAmount){
        require(!(tokenXAmount == 0 && tokenYAmount == 0), "invalid input");
        require(!(tokenXAmount != 0 && tokenYAmount != 0), "invalid input");

        tokenX_in_LP = tokenX.balanceOf(address(this));
        tokenY_in_LP = tokenY.balanceOf(address(this));
        
        if(tokenXAmount != 0){
            **outputAmount = tokenY_in_LP * ((tokenXAmount * 10 ** 18) * 999 / 1000) / ((tokenX_in_LP * 10 ** 18) + ((tokenXAmount * 10 ** 18) * 999 / 1000));**

            require(outputAmount >= tokenMinimumOutputAmount, "minimum ouput amount check failed");
            tokenY_in_LP -= outputAmount ;
            tokenX_in_LP += tokenXAmount;
            tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
            tokenY.transfer(msg.sender, outputAmount );
        }
        else{
            outputAmount = tokenX_in_LP * (tokenYAmount * 999 / 1000) / (tokenY_in_LP + (tokenYAmount * 999 / 1000));

            require(outputAmount >= tokenMinimumOutputAmount, "minimum ouput amount check failed");
            tokenX_in_LP -= outputAmount;
            tokenY_in_LP += tokenXAmount;
            tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
            tokenX.transfer(msg.sender, outputAmount);
        }
        return outputAmount;
    }
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 201277)
Logs:
  output: 998
```

이전의 해결방안과 마찬가지로 10 ** 18을 분자 분모에 곱해줘서 round down을 최소화 시킬 수 있다.

## removeLiquidity Round Down
#### Dex.sol:removeLiquidity:59-60
**************Low (파급력: Low, 공격 난이도:Low)**************

```solidity
function removeLiquidity(uint256 LPTokenAmount, uint256 minimumTokenXAmount, uint256 minimumTokenYAmount) public returns(uint, uint){
        require(balanceOf(msg.sender) >= LPTokenAmount, "RemoveLiquidity exceeds balance check error");
        tokenX_in_LP = tokenX.balanceOf(address(this));
        tokenY_in_LP = tokenY.balanceOf(address(this));
        **uint remove_tokenXAmount_in_LP = tokenX_in_LP * LPTokenAmount / totalSupply();**
        **uint remove_tokenYAmount_in_LP = tokenY_in_LP * LPTokenAmount / totalSupply();**
        require(remove_tokenXAmount_in_LP >= minimumTokenXAmount, "RemoveLiquidity minimum return error");
        require(remove_tokenYAmount_in_LP >= minimumTokenYAmount, "RemoveLiquidity minimum return error");
        _burn(msg.sender, LPTokenAmount);
        tokenX_in_LP -= remove_tokenXAmount_in_LP;
        tokenY_in_LP -= remove_tokenYAmount_in_LP;
        tokenX.transfer(msg.sender, remove_tokenXAmount_in_LP);
        tokenY.transfer(msg.sender, remove_tokenYAmount_in_LP);
        return (remove_tokenXAmount_in_LP, remove_tokenYAmount_in_LP);
    }
```

각 토큰의 리저브에서 LP토큰 지분을 계산하는 과정에서 decimal을 곱해주지 않고 그냥 계산하기 때문에 라운드 다운이 생길 수 있다.

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
[PASS] testRoundDownRemovLiquidity() (gas: 205422)
Logs:
  LP1: 972000000000
  EX: 57176470588235294117
  rx: 57176470588000000000
  ry: 57176470588000000000
```

위와같이 기대하던 값인 EX에 못미치는 round down된 rx, ry가 리턴된다.

#### 해결방안

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) public returns (uint256 LPTokenAmount){
        require(!(tokenXAmount == 0 && tokenYAmount == 0), "AddLiquidity invalid initialization check error - 1");
        require(tokenXAmount != 0, "AddLiquidity invalid initialization check error - 2");
        require(tokenYAmount != 0, "AddLiquidity invalid initialization check error - 3");
        require(tokenX.allowance(msg.sender, address(this)) >= tokenXAmount, "ERC20: insufficient allowance");
        require(tokenY.allowance(msg.sender, address(this)) >= tokenYAmount, "ERC20: insufficient allowance");
        require(tokenX.balanceOf(msg.sender) >= tokenXAmount, "ERC20: transfer amount exceeds balance");
        require(tokenY.balanceOf(msg.sender) >= tokenYAmount, "ERC20: transfer amount exceeds balance");
        
        tokenX_in_LP = tokenX.balanceOf(address(this));
        tokenY_in_LP = tokenY.balanceOf(address(this));

        if(first_LP){
            **LPTokenAmount = Math.sqrt((tokenXAmount + tokenX_in_LP) * (tokenYAmount + tokenY_in_LP));**
            first_LP = false;
        }
        else{
            require(tokenX_in_LP * tokenYAmount == tokenY_in_LP * tokenXAmount, "AddLiquidity imbalance add liquidity test error");
            LPTokenAmount = Math.min(
                totalSupply() * tokenXAmount / tokenX_in_LP,
                totalSupply() * tokenYAmount / tokenY_in_LP
            );
        }
        require(LPTokenAmount >= minimumLPTokenAmount, "AddLiquidity minimum LP return Error");
        _mint(msg.sender, LPTokenAmount);
        tokenX_in_LP += tokenXAmount;
        tokenY_in_LP += tokenYAmount;
        tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
        tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
        return LPTokenAmount;
    }
```

```solidity
[PASS] testRoundDownRemovLiquidity() (gas: 203555)
Logs:
  LP1: 972000000000000000000
  EX: 57176470588235294117
  rx: 57176470588235294117
  ry: 57176470588235294117
```

라운드 다운을 해결하기 위해서는 라운드 다운이 시작된 처음 LP token을 발행하는 곳에서 고쳐줘야 하는데, 원래 10 ** 18으로 나눠주던걸 없애주면 된다.
