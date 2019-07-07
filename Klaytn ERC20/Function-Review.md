# Klaytn을 활용한 ERC20 토큰 만들기

[Custom ERC20 Full Code](https://github.com/JohnsonRyu/CustomERC/blob/master/%EB%A5%98RC20.sol)

## 1. Function Review
```
contract ERC20Basic {
    function totalSupply() public view returns (uint256);
    function balanceOf(address who) public view returns (uint256);
    function transfer(address to, uint256 value) public returns (bool);
    event Transfer(address indexed from, address indexed to, uint256 value);
}
```
```
contract ERC20 is ERC20Basic {
    function allowance(address owner, address spender) public view returns (uint256);
    function transferFrom(address from, address to, uint256 value) public returns (bool);
    function approve(address spender, uint256 value) public returns (bool); 
    event Approval(address indexed owner, address indexed spender, uint256 value);
}
```

#### 1-1. totalSupply()

`uint256 totalSupply_;`

```
function totalSupply() public view returns (uint256) {
    return totalSupply_;
}
```

> 총 발행량을 나타냅니다. 토큰의 발행량을 설정 하는 방법은 해당 변수를 직접 세팅하거나, mint Contract를 사용한 방법이 있습니다.

<br/>

**EX1) 생성자를 통한 총 발행량 세팅**

```
constructor (uint256 _totalSupply) public
    totalSupply_ = _totalSupply;
}
```

<br/>

**EX2) mintContract를 통한 발행량 세팅**

```
contract MintableToken is StandardToken, Ownable {

    function mint(address _to, uint256 _amount) public returns (bool) {
        totalSupply_ = totalSupply_.add(_amount);
        balances[_to] = balances[_to].add(_amount);
    
        emit Mint(_to, _amount);
        emit Transfer(address(0), _to, _amount);
    
        return true;
    }
}
```

> 초기에는 소각을 진행 하더라도 totalSupply 변수를 변경하지 않고 유효하지 않은 주소로 송금하는 방법을 사용했습니다. 그렇게 되면 totalSupply()를 호출하여 정보제공을 하는 다양한 사이트 혹은 유저들이 소각 여부를 판단하기가 힘들었습니다.

#### 1-2. balanceOf(address _who)

`mapping(address => uint256) balances;`

```
function balanceOf(address _who) public view returns (uint256) {
    return balances[_who];
}
```

> address `_who`의 잔액을 조회합니다. decimals가 필터 되지 않고 출력 되기 때문에 `n개 * 10**uint(decimals);` 형태로 출력됩니다. 유저가 자신의 잔액을 보는 거래소, 지갑 등에서는 해당 변수에 decimals를 나누어주는 작업을 하지만, Klaytn IDE를 통하여 출력하게 되면 필터되지 않은 자연의 상태로 출력 됩니다.

#### 1-3. transfer(address to, uint256 value)

`mapping(address => uint256) balances;`

```
function transfer(address _to, uint256 _value) public returns (bool) {
    require(_to != address(0));
    require(_value <= balances[msg.sender]);
  
    balances[msg.sender] = balances[msg.sender].sub(_value);
    balances[_to] = balances[_to].add(_value);
  
    emit Transfer(msg.sender, _to, _value);
    return true;
}
```

> address `to`에게 uint256 `_value`만큼의 토큰을 전송합니다. balanceOf와 마찬가지로 decimals가 필터되지 않은 값에서 증가하고, 감소하기 때문에 호출시 `_value`에 decimals를 붙여야 합니다.

<br/>

**EX1) `Decimals : 18` 일 때 12개 전송**

```
transfer(_to, 12000000000000000000);
```

**EX2) `Decimals : 18` 일 때 4231.234개 전송**

```
transfer(_to, 4231234000000000000000);
```

> 예전에는 이런 방식이 너무 귀찮아서, transfer 함수 자체에 `_value * 10**uint(decimals);`를 넣어 인자 값을 편하게 전달하려고 했지만 각종 지갑 사이트에서도 이미 해당 작업을 자신들이 해주기 때문에 중복 되는 경우가 발생 되었습니다.

#### 1-4. approve(address spender, uint256 value)

`mapping (address => mapping (address => uint256)) internal allowed;`

```
function approve(address _spender, uint256 _value) public returns (bool) {
    allowed[msg.sender][_spender] = _value;

    emit Approval(msg.sender, _spender, _value);

    return true;
}
```

> address `spender`에게 uint256 `value`만큼 토큰 권한을 양도 합니다. transfer와 다르게 잔액이 즉시 증가되고 감소되는 것이 아닌, 권한만 전달 하고 추후 transferFrom 함수를 통해 transfer를 진행 합니다.

#### 1-5. allowance(address owner, address spender)

`mapping (address => mapping (address => uint256)) internal allowed;`

```
function allowance(address _owner, address _spender) public view returns (uint256) {
    return allowed[_owner][_spender];
}
```

> address `_owner`가 address `_spender`에게 권한을 양도한 토큰의 수를 확인하는 balanceOf와 같은 함수 입니다.

#### 1-6. transferFrom(address from, address to, uint256 value)

`mapping (address => mapping (address => uint256)) internal allowed;`

```
function transferFrom(address _from, address _to, uint256 _value) public returns (bool) {
    // 보내는 주소가 유효한가?
    require(_to != address(0));
    // 보내는 주소에 충분한 잔액이 있는가?
    require(_value <= balances[_from]);
    // 보내는 사람이 받는 사람에게 _value 혹은 그 이상 양도를 한 상태인가?
    require(_value <= allowed[_from][msg.sender]);
    // 조건이 성립되었다면 from의 잔액을 빼고,
    balances[_from] = balances[_from].sub(_value);
    // to의 잔액에 더한다.
    balances[_to] = balances[_to].add(_value);
    // 그리고 양도된 양에서 _value를 뺀다.
    allowed[_from][msg.sender] = allowed[_from][msg.sender].sub(_value);

    emit Transfer(_from, _to, _value);

    return true;
}
```

> address `_from`가 address `_to`에게 uint256 `_value`만큼 토큰을 전송합니다. `transfer(address _to uint256 _value)`와 다른 점은 `from`입니다. 말 그대로 제 3자가 양도 된 토큰에 한하여 대신 전송을 해줄 수 있음을 의미합니다.

## transfer 세트와 transferFrom 세트는 비슷 비슷 한데. 이걸 어디에 써요?

**상점 Contract를 구현**

체력 포션 500 Token / 마나 포션 400 Token 일 때,
유저는 포션을 받기 위하여 상점 Contract에 자신이 원하는 수량 만큼의 Token을 전송해야합니다.
하지만, Contract는 유저의 주소에서 token을 가지고 갈 수 없습니다.(transfer함수를 호출할 수 없습니다.)

`transfer(address _to, uint256 _value);`

transfer 함수는 자기 자신의 토큰을 본인만 송금을 할 수 있는 함수이기 때문이죠.

그렇기 때문에 Contract는 유저가 물건을 구매하려고 할 때, 혹은 그 전에 유저의 token 권한을 `approve(address spender, uint256 value)`를 통하여 위임 받은 후,
유저가 포션을 구매하려고 하려고 `buyPotion(uint _HP, uint _MP);`을 호출 했을 때, 위임 받은 토큰을 `transferFrom(address _from, address _to, uint256 _value)`을 통하여 원하는 주소로 송금합니다.