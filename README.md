### 모든 문서는 표준 구현체인 eth-infinitism을 기반으로 작성함.
주석을 많이 달아 문서의 길이가 길어질 수도 있음. 단, 가독성은 보장함.

```
dockers/workdir/bundler.config.json

{
  "mnemonic": "mnemonicfile.txt",
  "network": "https://goerli.infura.io/v3/f40be2b1a3914db682491dc62a19ad43",
  "beneficiary": "0x90F8bf6A479f320ead074411a4B0e7944Ea8c9C1",
  // fee를 receive받을 주소. 프로젝트의 자금줄 주소.
  "port": "80",
  "helper": "0x214fadBD244c07ECb9DCe782270d3b673cAD0f9c",
  "entryPoint": "0x602aB3881Ff3Fa8dA60a8F44Cf633e91bA1FdB69",
  // 우리가 아는 그 entrypoint. config에다가 넣어놓은 이유는 새 버전 나올때마다 갈아끼우기 용이해서인듯?? 
  "minBalance": "0",
  "gasFactor": "1"
}
```


```
// packages/bundler/src/BundlerServer.ts

  async handleMethod (method: string, params: any[]): Promise<any> {
  //타입스크를 사용해서 method에다가 호출할 함수를 넣어서 switch하고 params를 array로 넣어서 매개변수를 넣나봄. 
    let result: any
    switch (method) {
      case 'eth_chainId':
        // eslint-disable-next-line no-case-declarations
        const { chainId } = await this.provider.getNetwork()
        result = chainId
        break
      case 'eth_supportedEntryPoints':
        result = await this.methodHandler.getSupportedEntryPoints()
        break
      case 'eth_sendUserOperation': // useroperation, entrypointaddress를 받아감
        result = await this.methodHandler.sendUserOperation(params[0], params[1])
        break
 and more....
```

```
  // add userOp into the mempool, after initial validation.
  // replace existing, if any (and if new gas is higher)
  // revets if unable to add UserOp to mempool (too many UserOps with this sender)
  // packages/bundler/src/modules/MempoolManager.ts

/*
	이거 진짜 솔리디티 아니네 미친;; adduserop함수를 우리가 호출해서 멤풀에 던지는 식으로 이루어질 거 같은데, 이거 DB가 어디에 있는지 잘 모르겠음. tendermint나 geth는 kvstore기반 db인터페이스를 모든 노드들한테 제공하고 분산해서 저장하게끔 해놓긴 했는데. 얘는 사실 특정 기업이 운영하는 거라서 분산 자체가 필요없이 하나의 서버에 다 던지는 방식으로 이루어질 수도 있거든요??? 근데 그거랑 별개로 '체인'을 만들어야 분산원장 시스템이 돌아가는데 해당 userOp라는 TX용 쿼리를 아에 영구저장할 필요가 없잖아요. 그래서 크게 데이터 저장공간이 필요하지는 않을 거 같음. 이건 저희 클라이언트 정책 따라서 벨리데이터가 있을지 없을지 유무로 판단해야 할거 같아요. (근데 타입스크?? 라서 과연 여기까지 오딧을 진행할지는 미지수.)
일단 표준에 추가되어있긴 하니까 포맷은 거기서 거기일 듯??? 

*/
  addUserOp (userOp: UserOperation, userOpHash: string, prefund: BigNumberish, senderInfo: StakeInfo, referencedContracts: ReferencedCodeHashes, aggregator?: string): void {
    const entry: MempoolEntry = {
      userOp,
      userOpHash,
      prefund,
      referencedContracts,
      aggregator
    }
    const index = this._findBySenderNonce(userOp.sender, userOp.nonce)
    if (index !== -1) {
      const oldEntry = this.mempool[index]
      this.checkReplaceUserOp(oldEntry, entry)
      debug('replace userOp', userOp.sender, userOp.nonce)
      this.mempool[index] = entry
    } else {
      debug('add userOp', userOp.sender, userOp.nonce)
      this.entryCount[userOp.sender] = (this.entryCount[userOp.sender] ?? 0) + 1
      this.checkSenderCountInMempool(userOp, senderInfo)
      this.mempool.push(entry)
    }
    this.updateSeenStatus(aggregator, userOp)
  }
```

```
contract EntryPoint {
    // ...
    function handleOps( // sigAggregation 필요없으면 이거 호출함
        UserOperation[] calldata ops,
        address payable beneficiary
    ) public nonReentrant {
        uint256 opslen = ops.length;
        UserOpInfo[] memory opInfos = new UserOpInfo[](opslen);

        unchecked {
            for (uint256 i = 0; i < opslen; i++) {
                UserOpInfo memory opInfo = opInfos[i];
                (
                    uint256 validationData,
                    uint256 pmValidationData
                ) = _validatePrepayment(i, ops[i], opInfo);

// *_validatePrepayment*
// validation에 들어가는 가스비가 사전에 검증에 사용되는 가스비 상한선을 넘는지 안넘는지 체크함
// _validateAccountPrepayment라는 내부함수를 호출하는데, 여기서 만약 AA 계정이 없으면 만들어줌
                
                _validateAccountAndPaymasterValidationData(
                    i,
                    validationData,
                    pmValidationData,
                    address(0)
                );
// *_validateAccountAndPaymasterValidationData*
// 만약 validationData가 만료되었으면(expired) revert함.
            }

            uint256 collected = 0;
            emit BeforeExecution();

            for (uint256 i = 0; i < opslen; i++) {
                collected += _executeUserOp(i, ops[i], opInfos[i]);
            }
// *_executeUserOp*
// 개별 UserOperation을 실행하는 함수.
// 직설적으로 말하면 UserOperation의 calldata를 월렛 컨트랙트에서 실행하는 거고
//쉽게 이야기하면 TX실행하는 거.
//해당 함수도 여러가지 내부함수를 가지고는 있는데, 일단은 가장 중요한 innerHandleOp함수만 먼저 가져왔어요

            _compensate(beneficiary, collected);
        }
    }
    // ...
}
```

```
contract EntryPoint { 
    // ...
    function innerHandleOp(
        bytes memory callData,
        UserOpInfo memory opInfo,
        bytes calldata context
    ) external returns (uint256 actualGasCost) {
        uint256 preGas = gasleft();
        
        require(msg.sender == address(this), "AA92 internal call only");
		// 얘가 다른 같은 컨트랙트에서 다시 호출되는 함수거든요?? 근데 왜 external쓰고 
        // this.innerHandleOps 이런 식으로 호출한건지 이해가 안감 그냥 internal 쓰면 안되나?


        // ...
        // check if handleOps was called with gas limit too low. abort entire bundle
        // ...
        // executes UserOperation on the wallet smart contract account and emits event if the operation reverts
            bool success = Exec.call(mUserOp.sender, 0, callData, callGasLimit);
        // ...
        // executes post operation
            return _handlePostOp(0, mode, opInfo, context, actualGas);
        // ...
    }
}
```

```
contract EntryPoint {

    // *handleAggregatedOps*
    // sigAggregation이 필요한 경우 호출되는 함수인데, 이게 그거에요 Batch verification....
    
    function handleAggregatedOps(
        UserOpsPerAggregator[] calldata opsPerAggregator,
        address payable beneficiary
    ) public nonReentrant {

        uint256 opasLen = opsPerAggregator.length;
        uint256 totalOps = 0;
        for (uint256 i = 0; i < opasLen; i++) {
            UserOpsPerAggregator calldata opa = opsPerAggregator[i];
            UserOperation[] calldata ops = opa.userOps;
            IAggregator aggregator = opa.aggregator;

            //address(1) is special marker of "signature error"
            require(
                address(aggregator) != address(1),
                "AA96 invalid aggregator"
            );

            if (address(aggregator) != address(0)) {
                // solhint-disable-next-line no-empty-blocks
                try aggregator.validateSignatures(ops, opa.signature) {} catch {
                    revert SignatureValidationFailed(address(aggregator));
                }
            }

            totalOps += ops.length;
        }

        // ...
        for (uint256 a = 0; a < opasLen; a++) {
            // ... _validatePrepayment
            // ... _validateAccountAndPaymasterValidationData
        }

        // ...
        for (uint256 a = 0; a < opasLen; a++) {
            // ... _executeUserOp
        }
        // ...
        _compensate(beneficiary, collected);
    }
    // ...
}
```

### ERC-4337 논스

#### - 왜 필요한가?

- 재실행 공격 방지용
- 트랜잭션 순서 보장(각 명령당 순서를 보장할 무언가가 필요함)
- TX 고유성 확보 및 트래킹 용이 (layerzero stargate tracker같은거 만들기 쉬움)
#### - 어떻게 쓰이고 있는가?

v0.6기준 Entrypoint가 NonceManager를 상속받아 extension식으로 사용되는 거 같은데... 논스를 직접 관리하나봄.
Q. 그럼 각 계정별로 논스를 관리하는 건가?
Q. 스토리지값 안부족하나??? 

#### - 이전 버전과 달라진 점은?

v0.4버전에서는 지갑이 해당 논스를 관리했었음.



## wallet architecture

For that, an important design goal is to replicate the key property of EOAs in which users do not need to perform any custom action to create their wallets. They can simply generate an address locally and immediately start accepting funds. The wallet creation is thus done by a “factory” contract, with wallet-specific data, which uses [CREATE2](https://eips.ethereum.org/EIPS/eip-1014) to create the wallet in a counterfactual (deterministic) address.

Wallet, 즉 aa지갑은 기본적으로 create2함수를 이용해서 만들어진다. 

```
* User Operation struct  
* @param sender - The sender account of this request.  
* @param nonce - Unique value the sender uses to verify it is not a replay.  
* @param initCode - If set, the account contract will be created by this constructor/
```

UserOperation 구조체 필드 중에 `initCode`가 있는 게 보일 텐데, 해당 initCode의 필드값에 데이터가 있을 경우 entrypoint contractdptj getSenderAddress()를 호출하는데

```
// contracts/core/EntryPoint.sol line 450

    /// @inheritdoc IEntryPoint
    function getSenderAddress(bytes calldata initCode) public {
        address sender = senderCreator().createSender(initCode);
        revert SenderAddressResult(sender);
    }
```

해당 함수는 createsender함수를 호출해서 만든 다음에, revert를 시키...네????? 이렇게 하면 반환값을 더 가스 efficient하게 받을 수 있어서 그런가 보다. try-catch구문을 사용하면 쉽게 데이터 처리가 가능하니까.
무튼 다시 돌아와서 createSender에서는 뭘 하느냐.

```
contract SenderCreator {
    /**
     * Call the "initCode" factory to create and return the sender account address.
     * @param initCode - The initCode value from a UserOp. contains 20 bytes of factory address,
     *                   followed by calldata.
     * @return sender  - The returned address of the created account, or zero address on failure.
     */
    function createSender(
        bytes calldata initCode
    ) external returns (address sender) {
        address factory = address(bytes20(initCode[0:20]));
        bytes memory initCallData = initCode[20:];
        bool success;
        /* solhint-disable no-inline-assembly */
        assembly {
            success := call(
                gas(),
                factory,
                0,
                add(initCallData, 0x20),
                mload(initCallData),
                0,
                32
            )
            sender := mload(0)
        }
        if (!success) {
            sender = address(0);
        }
    }
}
```

맨날 보는 그거한다. create2이용해서 어셈으로 계정 만드는 거...근데 여기서 중요한 건 initCode를 진짜 initbytecode로 사용한다는 거다. 이게 무슨 소리냐, 해당 initCode에 우리가 어떤 데이터를 끼워넣든 간에 그게 런타임바이트코드로 들어간다는 소리임(!!!!!!!!) 이거 바이트코드 검수 안해도 되냐???? 악의적인 AA를 만드는 게 가능할 거 같은데???

This is interesting, because, although [Infinitism's repository](https://github.com/eth-infinitism/account-abstraction) provides a [BaseAccount](https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/core/BaseAccount.sol) implementation of the IAccount interface and a [SimpleAccount](https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/samples/SimpleAccount.sol) sample contract (created by [SimpleAccountFactory](https://github.com/eth-infinitism/account-abstraction/blob/develop/contracts/samples/SimpleAccountFactory.sol)), none of these are actual requirements of the specification. Developers are free to customize the wallet and factory implementation in any way they choose, provided that they correctly perform the necessary validations according to the ERC.

사실 infinitism repository, 즉 다오 커뮤니티가 빌딩중인 해당 erc에선 인터페이스를 제공해주기는 하는데, 해당 인터페이스는 표준이 아니라고 함;;;; 이게 문제인게, aa만드는 걸 전적으로 유저 책임으로 돌린다는 소리와도 같은데, 이러면 aa지원해주는 기업이 사기치고 initcode에 백도어 심어두면 할말이 없거든요... 그렇다고 꼬우니 내가 직접 userOp날린다 하기에는 batch로 인한 이점을 가져오지 못하는;;;일이 발생함. 즉, AA는 결국 중앙화된 entity에 대한 의존도가 굉장히 높다는 거고, 이게 맞냐??????????

```
abstract contract BaseAccount is IAccount {
    // ...

    /**
     * Sends to the entrypoint (msg.sender) the missing funds for this transaction.
     * SubClass MAY override this method for better funds management
     * (e.g. send to the entryPoint more than the minimum required, so that in future transactions
     * it will not be required to send again).
     * @param missingAccountFunds - The minimum value this method should send the entrypoint.
     *                              This value MAY be zero, in case there is enough deposit,
     *                              or the userOp has a paymaster.
     */
    function _payPrefund(uint256 missingAccountFunds) internal virtual {
        if (missingAccountFunds != 0) {
            (bool success, ) = payable(msg.sender).call{
                value: missingAccountFunds,
                gas: type(uint256).max
            }("");
            (success);
            //ignore failure (its EntryPoint's job to verify, not account.)
        }
    }
}
```
그러면 entrypoint에서는 어떤 동작이 일어나냐. 일단 해당 코드, `_payPrefund`를 호출해서 가스로 쓸 이더를 가져옴. 페이마스터가 있으면 해당 계정에서 가져올 텐데 어떤 경우에 페이마스터 컨트랙트를 호출하는지 등의 움직임은 나중에 보면 될듯??

이거 말고도 nonce검증이랑 validatePrepayment도 진행하는 거 같은데, 나중에 추가하는 걸로.

## Wallet execution
이제 검증도 끝났으니 wallet execution을 할 차례.
```
contract SimpleAccount is BaseAccount, TokenCallbackHandler, UUPSUpgradeable, Initializable {
    // ...

    /**
     * execute a transaction (called directly from owner, or by entryPoint)
     */
    function execute(address dest, uint256 value, bytes calldata func) external {
        _requireFromEntryPointOrOwner();
        _call(dest, value, func);
    }

    /**
     * execute a sequence of transactions
     * @dev to reduce gas consumption for trivial case (no value), use a zero-length array to mean zero value
     */
    function executeBatch(address[] calldata dest, uint256[] calldata value, bytes[] calldata func) external {
        _requireFromEntryPointOrOwner(); // owner이거나 entrypoint에서 직접 interact하는 경우만 허용함
        require(dest.length == func.length && (value.length == 0 || value.length == func.length), "wrong array lengths");
        if (value.length == 0) {
            for (uint256 i = 0; i < dest.length; i++) {
                _call(dest[i], 0, func[i]);
            }
        } else {
            for (uint256 i = 0; i < dest.length; i++) {
                _call(dest[i], value[i], func[i]);
            }
        }
    }
    
    // ...
}
```
해당 wallet 안에 있는 내부함수임. 근데 여기 보면 owner이거나 entrypoint일때만 허용한다고 하는데, 해당 계정 스토리지값이 달라지면 큰일나는거 아닌가? 아님 owner주소를 다른 걸로 바꾸거나,,,등등. 여기 upgradable함수가 존재한다고 하던데, 제대로 initalize안하면 큰일날듯.


## Paymaster

Thanks to this flexibility in the specification, innovative models like transaction sponsoring, subscription-based gas payment, or dynamic fees can be implemented.

사실상 aa지갑 만드는 회사는 이거 덕분에 수수료 모델을 굉장히 동적으로 작성할 수 있게 되는 거 같음.
페이마스터 주소는 userOp의 파라미터 필드값이 넣어지나봄. 

#### validation stage에서의 paymaster
페이마스터는 verificationGasLimit보다 3배 이상 많은 가스를 보유하고 있어야 하는데, 이는 postOp같이 트랜잭션을 시뮬레이션하거나 실행하는 경우의 수가 최대 3번이여서 그런거 같음.
- `postOp`는 최대 두 번 호출될 수 있습니다:
    1. UserOperation 실행 후
    2. Paymaster의 첫 `postOp`가 실패했을 때
- 이는 Paymaster에게 스폰서링 실패를 알리고, 적절한 조치를 취할 기회를 줍니다.

validatePaymasterUserOp 함수:

- 이 함수는 Paymaster에서 호출되어 UserOperation에 대한 지불 의사를 확인합니다.
해당 함수는 유저가 해당 페이마스터를 사용하는지 여부를 검증함과 동시에, 사용자의 수수료 처리 모델에 맞춰 알고리즘을 작성할 수 있는 부분이다 보니 개발사의 재량에 따라 달려있더라고요. 아마 이건 클라이언트측 보면 될듯.

- 예시: a) 특정 사용자 그룹의 수수료를 보조하는 서비스 제공자: 계정이 허용 목록에 있는지 확인합니다. b) ERC-20 토큰으로 지불을 허용하는 Paymaster: 사용자의 토큰 잔액을 확인하고, 토큰을 차감한 후 이더로 지불합니다.

#### Execution stage에서의 paymaster

가장 먼저\?? 호출되는 함수는 userOperation 함수.

```
If `validatePaymasterUserOp` returns a non-empty context, `handleOps` calls `postOp` on the paymaster after the operation has been executed, during `_handlePostOp`.
```
validatePaymasterUserOp에서 useroperation의 유효성을 검사하고, context라는 반환값에 따라 처리를 도맡아서 해줌.

userOperation함수가 호출된 다음에 entrypoint에 의해 postOp함수가 다시 호출되는데, 이는userOperation함수 호출 이후 추가적인 로직을 수행할 수 있게 해줌.

paymaster는 validationUserOp에서 한번, postOp에서 한번 가격을 확인하는데, 전자의 경우 balance가 없으면 execute를 안하기 위함이고 후자의 경우 최종 비용 계산 때문임.
그래서 플래시론에 기반한 dos어택이 들어올 수도 있는데, 
- 예상치 못한 상황 발생:
    - UserOperation이 플래시론을 수행합니다.
    - 이로 인해 Paymaster가 의존하는 유동성 풀의 균형이 깨집니다.
    - UserOperation은 성공했지만, Paymaster의 `postOp`가 실패(revert)합니다.
- EntryPoint의 대응:
    - EntryPoint는 Paymaster를 다시 호출합니다.
    - `postOpReverted` 모드로 Paymaster에게 상황을 알립니다.
    - Paymaster는 전체 번들을 취소할지 결정할 수 있습니다.
    - 이 결정은 평판 시스템에 영향을 미칩니다.
이런 시나리오로 대응할 수 있음.
postOpReverted 호출이 if문으로 있을 거 같고 mode는 enum을 쓸지 아님 bool값을 쓸지는 미지수. 구현체 봐야해요.

**이러한 post-reversion 실패에 대한 책임은 Paymaster에게 묻기 때문에, 이는 paymaster의 평판점수하고도 직결되는 문제인 거 같아요. 정확한 점수 측정은 EntryPoint에 있는 거 같고. 평판깎이면 쓰로틀링 들어가는듯.(일정 시간동안 4337 entrypoint를 이용할 수 없다거나...등등. entrypoint 코드 까보면 블랙리스트 코드 나올지도???)**

## paymaster가 평판을 유지하는 방법은 거래를 잘 끝내거나, 스테이킹을 하는 방법밖에 없는듯...? 근데 이 스테이킹하는거 이더리움 노드에다가 리스테이킹하면 안되나 어짜피 락업일텐데...??? paymaster에게 스테이킹 보상 다시 돌려주면 되고. <-EIP제안 요망



## Factory
계정 팩토리도 번들에서 tx보낼때 같이 보내나 ㅇㅇ

- 계정 팩토리 식별:
    - initCode의 첫 20바이트는 계정 팩토리의 주소입니다.
    - 이를 통해 어떤 팩토리가 사용되는지 직접적으로 알 수 있습니다.
- EntryPoint의 역할:
    - EntryPoint는 initCode를 해석하여 적절한 팩토리를 호출합니다.
    - 팩토리 주소가 유효한지 확인합니다.
- 기존 계정 vs 새 계정:
    - 기존 계정의 경우 initCode는 비어있습니다.
    - 새 계정 생성 시에만 initCode가 사용됩니다.

initCode에 추가해서 보내네요...진짜 중앙화인데??????이게맞냐;;;;;
