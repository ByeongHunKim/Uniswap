# v2-core
## 1. code analyze - UniswapV2Factory.sol
- 새로운 pair를 거래하는 컨트랙트를 생성해내고 관리하는 게이트웨이 같은 것
- 체크한 함수들
    - `createPair()`
    - `setFeeTo()` , `setFeeToSetter()`
      - fee 중의 일부, `0.05%`를 가져온다고 했는데, feeTo라는 address를 넣어놨다
      - 쓰이게 되면, 토큰 컨트랙트에서 거래되는 수수료의 일부가 가게 된다

```solidity
    /*
    tokenA, B의 주소를 인자로 받는다
    createPair()는 새로운 token exchange contract 즉, pair contract를 만들어내는 함수이다
    */
    function createPair(address tokenA, address tokenB) external returns (address pair) {
        // 당연히 두 토큰의 주소가 같으면 안된다 ( 서로 비교함 )
        require(tokenA != tokenB, 'UniswapV2: IDENTICAL_ADDRESSES');
        
        /*
        tokenB가 크면 A-B pair를 만들고, A가 크면 B-A pair를 만든다
        왜 이렇게 하냐면, A-B pair와 B-A pair가 같기 때문에, 이 둘을 구별하는 방법을 정해야 하는데 그 주소의 크기를 비교하고 토큰 pair의 순서를 정한다 
        Uniswap Info를 보면 WBTC-ETH pair가 있고, ETH-USDT pair가 있는데, 왜 ETH가 첫번쨰는 뒤에 있고, 두번째는 앞에 있는지는 이 코드 떄문
        */
        (address token0, address token1) = tokenA < tokenB ? (tokenA, tokenB) : (tokenB, tokenA);
      
        /*
        token0의 주소가 0인지 아닌지 검사. token1을 검사하지 않는 이유는 token0은 token1보다 주소가 작다
        그렇기 떄문에 token1은 무조건 0보다 크다. 그렇기 떄문에, token0만 검사하면 된다
        */
        require(token0 != address(0), 'UniswapV2: ZERO_ADDRESS');
        
        /*
        두 토큰 pair에 해당하는 컨트랙트가 이미 있는지 검사
        Factory가 토큰 pair를 관리하는 자료구조를 담고 있는 변수 이름이 getPair인데,
        이 변수에 token0, 1을 넣어보는 것.
        그랬을 때 값이 있으면 ( zero address(0) 가 아니면 ) 이미 있는 것
        만약에 있다면 revert 처리
        */
        require(getPair[token0][token1] == address(0), 'UniswapV2: PAIR_EXISTS'); // single check is sufficient
        
        /*
        import './UniswapV2Pair.sol';
        contract directory에 있었던 UniswapV2pair 컨트랙트의 코드를 바이트코드로 가져온다
        */
        bytes memory bytecode = type(UniswapV2Pair).creationCode;
        // 그리고 나서 token0, 1(abi)을 가지고 keccak256이라고 하는 해시 함수를 돌려서 salt를 하나 만든다
        bytes32 salt = keccak256(abi.encodePacked(token0, token1));
        
        /*
        create2 함수를 가지고 pair 컨트랙트를 만든다 ( 어셈블리어로 선언이 되어있다 )
        */
        assembly {
            pair := create2(0, add(bytecode, 32), mload(bytecode), salt)
        }
  
        /*
        token0, 1을 initialize(초기화) 해준다
        그러면 그 컨트랙트는 이제부터 token0, 1을 교환할 수 있는 그리고 그 풀을 형성하는 컨트랙트가 된다
        아까 위에서 getPair가 두 토큰을 교환하는 ( token exchange contract 의 주소를 담고 있는 ) 자료구조라고 했는데,
        여기다가 token0,1 쌍에 해당하는 element(요소)에 pair의 주소를 넣어주는 것
        모든 조합 getPair[token0][token1], getPair[token1][token0] 에 대해서 다 넣어주는데, 
        안정성을 위해서 하는 것 같다
        */      
        IUniswapV2Pair(pair).initialize(token0, token1);
        getPair[token0][token1] = pair;
        getPair[token1][token0] = pair; // populate mapping in the reverse direction
        
        // address[] public allPairs; 위에 선언한 변수에도 push를 해준다
        allPairs.push(pair);
        
        // 그 다음에 emit해서 새로 pair를 만들었다는 로그를 남겨준다
        emit PairCreated(token0, token1, pair, allPairs.length);
    }
```

### 1.1 UniswapV2Factory.sol 질문


## 2. code analyze - UniswapV2Pair.sol
- factory contract가 만들어낸 token pair contract
- 실제로 두 토큰을 거래하는 풀이자 거래소가 된다

```solidity
    // v2 pair는 factory가 생성하기 때문에, 먼저 constuctor에 factory 주소를 넣어준다
    // 이 pair contract를 배포한 컨트랙트가 factory contract일 것이기 때문에, msg.sender이다
    address public factory;
    constructor() public {
      factory = msg.sender;
    }
    
    /*
        factory에서 call(호출)을 하면 token0에는 token0 argument, token1에는 token1 argument를 대입해준다
  
    */
    
    function initialize(address _token0, address _token1) external {
      require(msg.sender == factory, 'UniswapV2: FORBIDDEN'); // sufficient check
      token0 = _token0;
      token1 = _token1;
    }
    
    /*
        계속해서 본인이 얼마나 들고 있는 지 추적함 
        하지만 추적이 항상 맞지 않은 이유는 실제 잔고(balance)는 token0,1 컨트랙트에 있기 때문에 잔고에 대한 정보는
        컨트랙트에 가야지 볼 수 있다. 여기서 기록하고 있는 토큰의 수량보다 토큰 컨트랙트에 가서 관찰했을 때 거기의 잔고가 더
        많다면, 돈을 받았다는 것을 알 수 있다. 즉 이 토큰 풀에 토큰이 입금 되었다는 것을 알 수 있다
        이런 정보를 보고 업데이트를 해주는 역할을 하는게 _update()이다
    */

    function _update(uint balance0, uint balance1, uint112 _reserve0, uint112 _reserve1) private {
        require(balance0 <= uint112(-1) && balance1 <= uint112(-1), 'UniswapV2: OVERFLOW');

        /*
            Uniswap을 price oracle로 쓰기 위한 로직
            각각의 price들이 유지된 시간의 가중치 만큼 더해서 평균을 내어 현재 가격을 결정
            
            uint public price0CumulativeLast;
            uint public price1CumulativeLast;
            이렇게 public으로 선언되어 있는 변수 값을 변경하기 떄문에, 다른 컨트랙트에서 이 Uniswap을 price oracle로 쓰려면
            위의 변수를 보면 된다. 그러면 시간에 따라서 달라지는 가격을 완충해서 볼 수 있기 때문에 다른 컨트랙트에서
            가격 정보를 받아올 수 있다
        */
        uint32 blockTimestamp = uint32(block.timestamp % 2**32);
        uint32 timeElapsed = blockTimestamp - blockTimestampLast; // overflow is desired
        if (timeElapsed > 0 && _reserve0 != 0 && _reserve1 != 0) {
        // * never overflows, and + overflow is desired
        price0CumulativeLast += uint(UQ112x112.encode(_reserve1).uqdiv(_reserve0)) * timeElapsed;
        price1CumulativeLast += uint(UQ112x112.encode(_reserve0).uqdiv(_reserve1)) * timeElapsed;
        }
        
        // reserve가 곧 받아온 값을 업데이트 해주는 건가?
        reserve0 = uint112(balance0);
        reserve1 = uint112(balance1);
        blockTimestampLast = blockTimestamp;
        emit Sync(reserve0, reserve1);
    }

    /*
      프로토콜 fee를 계산하는 공식을 담당하는 함수 
      백서를 좀 봐야한다..
    */
    function _mintFee(uint112 _reserve0, uint112 _reserve1) private returns (bool feeOn) {
        address feeTo = IUniswapV2Factory(factory).feeTo();
        feeOn = feeTo != address(0);
        uint _kLast = kLast; // gas savings
        if (feeOn) {
            if (_kLast != 0) {
                uint rootK = Math.sqrt(uint(_reserve0).mul(_reserve1));
                uint rootKLast = Math.sqrt(_kLast);
                if (rootK > rootKLast) {
                    uint numerator = totalSupply.mul(rootK.sub(rootKLast));
                    uint denominator = rootK.mul(5).add(rootKLast);
                    uint liquidity = numerator / denominator;
                    if (liquidity > 0) _mint(feeTo, liquidity);
                }
            }
        } else if (_kLast != 0) {
            kLast = 0;
        }
    }
    
    /*
        LP 토큰을 만들어내는 함수
        이 pair 컨트랙트가 기록하고 있는 자신의 자산의 양과 실제로 해당 token0 또는 1 컨트랙트에 기록되는
        자기 자산의 양이 다를 수 있다고 했다.
        case 1 : 만약에 token0,1 컨트랙트에 있는 내 자산의 양이 더 많다고 하면 mint 시켜준다
        case 2 : 반대로 더 적다면 burn 시켜준다
        그것을 보고 mint / burn 할 지 정하는 것이다
    */
    function mint(address to) external lock returns (uint liquidity) {
        (uint112 _reserve0, uint112 _reserve1,) = getReserves(); // gas savings
        
        // mint를 하기 위해서는 token0의 컨트랙트로 가서 balanceOf를 호출해서 내 잔고를 받아온다 ( 나머지 pair 토큰도 동일하게 잔고 체크 )
        uint balance0 = IERC20(token0).balanceOf(address(this));
        uint balance1 = IERC20(token1).balanceOf(address(this));

        /* 
        _reverve0(1)은 이 pair 컨트랙트에 기록하고 있는 잔고를 의미한다
        받아온 잔고에서 내가 기록한 잔고를 뺴면 내가 받은 amount가 계산된다
        */ 
        uint amount0 = balance0.sub(_reserve0);
        uint amount1 = balance1.sub(_reserve1);
        
        // 위 둘을 가지로 수수료 계산 ( 인플레이션이 되었기 때문 )
        bool feeOn = _mintFee(_reserve0, _reserve1);

        uint _totalSupply = totalSupply; // gas savings, must be defined here since totalSupply can update in _mintFee
        /*
        이 컨트랙트에 처음으로 pair를 넣는 사람이라면, LP share의 최소 단위 ( 1 LP token의 천배에 해당하는 돈이 burn )
        amount0 * amount1의 루트값(기하평균값..?) - minimum liquidity
        그리고 그 minimum  liquidity는 address0에게 생성된다
        그런데 address(0)의 소유주는 존재하지 않기 떄문에 보내지는 돈은 묶이게 된다
        결국 이 사람은 amount0 * amount1의 루트값(기하평균값..?) - minimum liquidity 이만큼 뺸 양의 liquidity를 가지고 가게 된다
        */ 
        if (_totalSupply == 0) {
            liquidity = Math.sqrt(amount0.mul(amount1)).sub(MINIMUM_LIQUIDITY);
            _mint(address(0), MINIMUM_LIQUIDITY); // permanently lock the first MINIMUM_LIQUIDITY tokens
        } else {
            /*
             새로 생성한 사람이 아니라면 그냥 인플레이션을 시킨다
             totalSupply * amount0 / reserve0 만큼을 생성히는데, 
             그 전에 token0,1 중에 더 적은 걸로 골라서 LP Token을 만들어 낸다
            */
            liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);
        }
        require(liquidity > 0, 'UniswapV2: INSUFFICIENT_LIQUIDITY_MINTED');
        
        // 모든 과정이 끝나면 지금 돈을 넣은 사람의 주소로 LP Token을 보내준다
        _mint(to, liquidity);
        
        // balance 업데이트 ( pair 컨트랙트에 잔고 기록 )
        _update(balance0, balance1, _reserve0, _reserve1);

        // kLast 업데이트 된 K 값 = 두 토큰 수량의 곱에 해당하는 K 값도 업데이트 해준다
        if (feeOn) kLast = uint(reserve0).mul(reserve1); // reserve0 and reserve1 are up-to-date

        // mint했다고 로그 남김
        emit Mint(msg.sender, amount0, amount1);
    }
```

### 2.1 UniswapV2Factory.sol 질문
- 왜 더 적은 걸로 골라서 LP TOKEN을 만들어 내는지 모르겠음
  - `liquidity = Math.min(amount0.mul(_totalSupply) / _reserve0, amount1.mul(_totalSupply) / _reserve1);`