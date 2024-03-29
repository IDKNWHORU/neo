# neo-mamba

## 빠른 시작
- `NodeJS 16.20`
- Linux 혹은 MaxOS 혹은 Windows 운영 체제
## 설치

설치 커맨드
```sh
$ npx create-react-app ./
```
### Pip를 이용한 neo-mamba 설치 
```sh
$ pip install neo-mamba
```
## 개발 코드 추가
소스 코드를 작성할 디렉터리를 추가한다.
```sh
$ touch src/NeolineSingleton.js
$ touch src/NeolineConnect.js
```
## 사용하지 않는 코드 삭제
```sh
$ rm src/App.test.js src/logo.svg src/setupTests.js
```
### NeolineSingleton.js 코드 추가
아래 코드를 `NeolineSingleton.js`에 추가한다.
```js
export const NeolineSignleton = {
    neoline: undefined,
    neolineN3: undefined
}
```
### NeolineConnect.js 코드 추가
아래 코드를 `NeolineConnect.js` 파일에 추가한다.
```js
import { NeolineSignleton } from "./NeolineSingleton";

/**
 * Neoline chrome 확장 프로그램이 설치되어 있어야 Neoline 라이브러리를 사용할 수 있는지 모니터링 할 수 있다.
 * 지금 모니터링하고 있는 이벤트 목록: NEOLine.NEO.EVENT.READY, NEOLine.N3.EVENT.READY
 * 
 */
export function initDapi(setIsLoading, setMessage) {
    const initCommonDapi = new Promise((resolve, reject) => {
        window.addEventListener('NEOLine.NEO.EVENT.READY', () => {
            // NeolineSignleton 객체의 neoline에 NEOLine API를 사용할 수 있게 설정한다.
            NeolineSignleton.neoline = new window.NEOLine.Init();
            if (NeolineSignleton.neoline) {
                resolve(NeolineSignleton.neoline);
            } else {
                reject('common dAPI method failed to load.');
            }
        });
    });
    const initN3Dapi = new Promise((resolve, reject) => {
        window.addEventListener('NEOLine.N3.EVENT.READY', () => {
            // NeolineSignleton 객체의 neolineN3에 NEOLineN3 API를 사용할 수 있게 설정한다.
            NeolineSignleton.neolineN3 = new window.NEOLineN3.Init();
            if (NeolineSignleton.neolineN3) {
                resolve(NeolineSignleton.neolineN3);
            } else {
                reject('N3 dAPI method failed to load.');
            }
        });
    });
    initCommonDapi.then(() => {
        console.log('The common dAPI method is loaded.');
        return initN3Dapi;
    }).then(() => {
        console.log('The N3 dAPI method is loaded.');
        setIsLoading(false);
        setMessage('');
    }).catch((err) => {
        console.log(err);
    })

    // Neoline API를 사용하는 설정이 되었는지 약 3초의 시간을 대기한다.
    // 설정되어 있지 않다면 Neoline Chrome 확장 프로그램이 설치되어있는지 물어보는 메시지를 UI에 출력한다.
    setTimeout(() => {
        if(NeolineSignleton.neoline === undefined) {
            setMessage('만약 Neoline이 설치되어 있지 않다면 https://chrome.google.com/webstore/detail/neoline/cphhlgmgameodnhkjdmkpanlelnlohao에서 Neoline을 설치해주세요')
        }
    }, 3000)
};
```
### App.js 코드 수정
`App.js` 파일의 내용을 아래 코드로 변경한다.
```js
import { useState } from 'react';
import './App.css';
import { NeolineSignleton } from './NeolineSingleton';
import { initDapi } from './NeolineConnect';

function App() {
  /**
   * UI에 데이터를 표현하기 위한 데이터의 상태를 정의하는 부분
   */
  const [isLoading, setIsLoading] = useState(true);
  const [isUserDataLoading, setIsUserDataLoading] = useState(false);
  const [message, setMessage] = useState('NeoLine 데이터를 불러오는 중입니다.');
  const [address, setAddress] = useState('');
  const [asset, setAsset] = useState([]);
  const [addressScriptHash, setAddressScriptHash] = useState('');

  const [targetAddress, setTargetAddress] = useState('');
  const [amount, setAmount] = useState(0);
  const [symbol, setSymbol] = useState('');

  /**
   * UI와 DApp간 상호 작용을 하기 위한 함수를 정의하는 부분
   */

  const getUserData = _ => {
    /**
     * Neoline chrome 확장 프로그램에 접속한 계정의 데이터를 가져온다
     */
    NeolineSignleton.neolineN3.getBalance().then(result => {
      if(!isUserDataLoading) {
        NeolineSignleton.neolineN3.AddressToScriptHash({ address: Object.keys(result)[0] }).then(({ scriptHash }) => { setAddressScriptHash(scriptHash) });
        setIsUserDataLoading(true);
        setAddress(Object.keys(result)[0]);
        setAsset(result[Object.keys(result)[0]]);
      }
      setSymbol(result[Object.keys(result)[0]][0].symbol);
    })
    .catch(err => {
      toastMessage(err.toString());
    })
  }

  /**
   * NEO, GAS, LUDIUM 토큰을 사용자가 입력한 주소로 보내는 기능
   */
  const send = _ => {
    switch (symbol) {
      case 'NEO': case 'GAS':
        NeolineSignleton.neolineN3.send({
          fromAddress: address,
          toAddress: targetAddress,
          asset: symbol,
          amount,
        }).then(getUserData)
          .catch(({ description }) => {
            toastMessage(description);
          });
        break;
      case 'LUDIUM':
        NeolineSignleton.neolineN3.invoke({
          scriptHash: 'ca3d9ddbc153c11dbd1abc0ff55e25b9fdcdbce3',
          operation: 'transfer',
          args: [
            { type: 'Address', value: address },
            { type: 'Address', value: targetAddress },
            { type: 'Integer', value: amount * 100000000 },
            { type: 'Any', value: null }
          ],
          signers: [{ account: addressScriptHash, scopes: 1 }]
        }).then(getUserData)
          .catch(({ description }) => {
            toastMessage(description);
          });
        break;
    }
  }

  /**
   * UI의 상태 값을 변경하는 함수
   */
  const toastMessage = message => {
    setMessage(message);
    setTimeout(() => { setMessage('') }, 2000);
  }

  const putAddress = ({ target }) => {
    setTargetAddress(target.value)
  }

  const putAmount = ({ target }) => {
    setAmount(target.value)
  }

  const changeSymbol = ({ target }) => {
    setSymbol(target.value);
  }

  /**
   * Neoline에 등록 된 사용자의 자산 정보를 UI에 표현한다.
   */

  const Balances = asset.map(({ amount, symbol }) => (
    <li key={symbol}>{symbol}: {amount}</li>
  ));

  /**
   * App 화면이 호출된 때 함수를 호출하는 부분
   * 시나리오
   * 1. Neoline 라이브러리 초기화
   * 2. Neoline에서 사용자 데이터 조회
   */
  if (isLoading) {
    initDapi(setIsLoading, setMessage);
  } else {
    if(!isUserDataLoading) getUserData();
  }

  return (
    <>
      {isLoading ? <p>{message}</p> : <div>
        {message === '' ? null : <><p>{message}</p> <hr /></>}
        <button onClick={getUserData}>데이터 불러오기</button>
        <p>{address}</p>
        <details open={asset.length > 0}>
          <summary>자산 목록</summary>
          <ul>
            {Balances}
          </ul>
        </details>
        <hr />
        <details open={asset.length > 0}>
          <summary>내 자산 보내기</summary>
          <ul>
            <li>
              <label htmlFor="address" style={{ marginRight: 30 }}>보내는 주소</label>
              <input type="text" id="address" onChange={putAddress} />
            </li>
            <li>
              <label htmlFor="quantity" style={{ marginRight: 78 }}>금액</label>
              <input type="number" id="quantity" onChange={putAmount} style={{ textAlign: 'end' }} />
              <select onChange={changeSymbol}>
                {asset.map(({ symbol }) => <option key={symbol} value={symbol}>{symbol}</option>)}
              </select>
            </li>
            <button onClick={send} style={{ width: 365, marginTop: 15 }}>send</button>
          </ul>
        </details>
      </div>
      }
    </>
  );
}

export default App;
```
## App, NeolineConnect, NeolineSingleton 모듈의 상호작용
![모듈 간 상호 작용 참고 자료 1](https://github.com/IDKNWHORU/neo/assets/49608580/5d85688f-7107-47c2-a0b4-426428c913a5)
![모듈 간 상호 작용 참고 자료 2](https://github.com/IDKNWHORU/neo/assets/49608580/a2a63812-de92-4970-a90e-aed3f6231be2)

## 실행
터미널에서 아래 명령어를 실행한다.
`npm run start`

아래와 같은 화면이 나타나는지 확인한다.
![실행 화면](https://github.com/IDKNWHORU/neo/assets/49608580/0e8c658e-579d-43f2-a074-4aae15109313)