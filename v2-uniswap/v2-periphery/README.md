# v2-periphery
## code analyze - UniswapV2Router02.sol
- Frontend 에서 사용한다
- 이거 말고 볼 파일은 없다
- 먼저 이걸 보고 -> V2core를 보면 된다
- 체크한 함수들
    - `_addLiquidity()`
    - `removeLiquidity()`
    - `_swap()`

```solidity
    function _addLiquidity(
        address tokenA, // tokenA의 contract address
        address tokenB, // tokenB의 contract address
        uint amountADesired, // 모르겠음
        uint amountBDesired, // 모르겠음
        uint amountAMin, // 모르겠음
        uint amountBMin // 모르겠음
    ) internal virtual returns (uint amountA, uint amountB) {
        // 1. constructor로 넣어준 factory contract address를 통해 factory contract에 접근해서 데이터를 얻는다. 
        // 1.1 tokenA 와 B로 구성된 pair가 있는 지 체크하고 없으면, 생성해준다
        if (IUniswapV2Factory(factory).getPair(tokenA, tokenB) == address(0)) {
            IUniswapV2Factory(factory).createPair(tokenA, tokenB);
        }
        
        // 2. 해당 풀의 a,b 가 얼만큼 들어있는 지 체크한다(?)
        (uint reserveA, uint reserveB) = UniswapV2Library.getReserves(factory, tokenA, tokenB);
        
        // 3. 둘 다 0 인 경우 초기 값을 세팅해주는 것 같은데, amountADesired amountBDesired 두 값을 파라미터로 받는데 잘 모르겠음
        if (reserveA == 0 && reserveB == 0) {
            (amountA, amountB) = (amountADesired, amountBDesired);
        } else {
            // 3.1 0이 아닌 경우는 quote함수를 통해서 어떤 값을 얻어내는 것 같다
            uint amountBOptimal = UniswapV2Library.quote(amountADesired, reserveA, reserveB);
            // 3.2 if 절과 else 절의 조건을 이해 못하였음
            if (amountBOptimal <= amountBDesired) {
                require(amountBOptimal >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
                (amountA, amountB) = (amountADesired, amountBOptimal);
            } else {
                uint amountAOptimal = UniswapV2Library.quote(amountBDesired, reserveB, reserveA);
                assert(amountAOptimal <= amountADesired);
                require(amountAOptimal >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
                (amountA, amountB) = (amountAOptimal, amountBDesired);
            }
        }
    }

    function removeLiquidity( // 포지션 종료의 의미로 이해
        address tokenA, // tokenA의 contract address
        address tokenB, // tokenB의 contract address
        uint liquidity, // 자신이 공급한 유동성?
        uint amountAMin, // 모르겠음
        uint amountBMin, // 모르겠음
        address to, // 보낼 수 있는 주소를 특정하게 해줌 ( 무조건 msg.sender면 두 번 보내니까 수수료를 아끼기 위해서 )
        uint deadline // 모르겠음
    ) public virtual override ensure(deadline) returns (uint amountA, uint amountB) {
        
        // 1. pair 정보를 얻어 내는 것?
        address pair = UniswapV2Library.pairFor(factory, tokenA, tokenB);
        
        // 2. 포지션 종료기 때문에, 이전에 공급한 유동성을 돌려 받는다
        IUniswapV2Pair(pair).transferFrom(msg.sender, pair, liquidity); // send liquidity to pair
        
        // 3. 2번과 같이 이해했는데, 이해가 100프로 되진 않았다
        (uint amount0, uint amount1) = IUniswapV2Pair(pair).burn(to);
        
        // 4. 잘 모르겠음
        (address token0,) = UniswapV2Library.sortTokens(tokenA, tokenB);
        
        // 5. 잘 모르겠음
        (amountA, amountB) = tokenA == token0 ? (amount0, amount1) : (amount1, amount0);
        
        // 6. 잘 모르겠음
        require(amountA >= amountAMin, 'UniswapV2Router: INSUFFICIENT_A_AMOUNT');
        require(amountB >= amountBMin, 'UniswapV2Router: INSUFFICIENT_B_AMOUNT');
    }

    // **** SWAP ****
    // requires the initial amount to have already been sent to the first pair
    function _swap(uint[] memory amounts, address[] memory path, address _to) internal virtual {
        
        // 기존에 v1은 무조건 ETH가 quote_token으로 풀로 만들어져서 다른 자산으로 swap하기 위해서는 2번 거쳐야 했다
        // v2에서부터는 이를 보완하기 위해서 ETH를 ERC-20처럼 사용하기 위해서 (transferFrom()) WETH를 사용한다 
        for (uint i; i < path.length - 1; i++) {
            
            // 잘 모르겠음
            (address input, address output) = (path[i], path[i + 1]);
            
            // 입력받은 주소를 통해 token0 주소를 정렬해준다 ( 정렬해주는 이유가 있었는데, 까먹었다 )
            (address token0,) = UniswapV2Library.sortTokens(input, output);
            
            // 잘 모르겠음
            uint amountOut = amounts[i + 1];
            
            // 잘 모르겠음
            (uint amount0Out, uint amount1Out) = input == token0 ? (uint(0), amountOut) : (amountOut, uint(0));
            
            // 잘 모르겠음
            address to = i < path.length - 2 ? UniswapV2Library.pairFor(factory, output, path[i + 2]) : _to;
            
            // 실제 스왑이 이루어지는 코드
            IUniswapV2Pair(UniswapV2Library.pairFor(factory, input, output)).swap(
                amount0Out, amount1Out, to, new bytes(0)
            );
        }
    }
```

## UniswapV2Router02.sol 질문
- `_addLiquidity()` 와 `addLiquidity()` 의 차이점에 대해서..
    -  `_` 가 붙은 건 직접 사용하는 것이고 안 붙은 건 interface 형태의 함수?