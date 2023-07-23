<template>
  <div class="container">
    <div class="top-right-buttons">
      <button v-if="!isConnected" id="connectWallet" class="connect-button" @click="enableEthereum">Connect wallet</button>
      <button v-else id="disconnectWallet" class="disconnect-button">{{ currentAccount.slice(0, 6)+'......'+currentAccount.slice(-6) }}</button>
    </div>
    <div v-if="transactionStatus" class="transaction-contrainer">
      <div class="transaction-status" >
        <div v-if="transactionPending" class="loader"></div>
        {{ transactionStatus }}
      </div>
      <div v-if="transactionLink">
        <a :href="transactionLink">Link</a>
      </div>
      <div>
        <button @click="closeWindow">Close</button>
      </div>
    </div>
    <div class="main-container">
      <div class="form-container">
        <form id="sendTransaction" class="mb-5" @submit.prevent="sendTransaction">
          <h2 class="mb-3">CANTILEVER</h2>
          <div class="bridge-container">
            <div class="destination-from">
              <div class="destination">
                From:
              </div>
              <div class="row">
                <select class="select select-80" v-model="selectedTokenFrom">
                  <option  v-for="option in filtredOptionsTokensFrom" :key="option.value" :value="option.value">
                    {{option.text }}
                  </option>
                </select>
                <select class="select select-20" v-model="selectedChainFrom">
                  <option v-for="option in optionsChains.map(({ value, text }) => ({ value, text: text.split(' ')[0] }))" :key="option.value" :value="option.value">
                    {{ option.text }}
                  </option>
                </select>
              </div>
            </div>
            <div class="destination-to">
              <div class="destination">
                To:
              </div>
              <div class="row">
                <select class="select select-100" v-model="selectedChainTo">
                  <option v-for="option in optionsChains" :key="option.value" :value="option.value">
                    {{option.text }}
                  </option>
                </select>
              </div>
            </div>
            <div class="amount">
              <div class="amount-and-balance">
                <div class="amount-div">
                  Total amount:<span class="balance-error">{{ balanceError }}</span>
                </div>
                <div class="balances-div">
                  Balances: <span>{{tokenBalance}}</span>
                </div>
              </div>
              <input class="input-amount" placeholder="0.0" v-model="amount"/>
            </div>
          </div>
            <button :disabled="!currentAccount" type="submit" class="btn btn-primary">Bridge</button>
        </form>
      </div>
    </div>
  </div>
</template>
  
  <script>
  import { ref, onMounted, watch } from 'vue';
  import detectEthereumProvider from '@metamask/detect-provider';
  import { ethers} from 'ethers';
  import abiUSDCPolygon from "@/ABI/polygonUSDC";
  import abiUSDCGnosis from "@/ABI/gnosisUSDC";
  import abicUSDCelo from "@/ABI/celocUSD";
  
  export default {
    setup() {
      let selectedTokenFrom = ref('USDC')
      let selectedChainFrom = ref('0x89')
      let selectedChainTo = ref('0x64')
      let amount = ref()
      
      const optionsTokens = ref([
        { value: 'USDC', text: 'USDC', chains: ['0x89', '0x64'] },
        { value: 'cUSD', text: 'cUSD', chains: ['0xa4ec'] },
        // { value: 'USDT', text: 'USDT', chains: ['0x44d'] },
      ])
      let filtredOptionsTokensFrom = ref([])
      const optionsChains = ref([
        { value: '0x89', text: 'Polygon USDC' },
        { value: '0x64', text: 'Gnosis USDC' },
        { value: '0xa4ec', text: 'Celo cUSD' },
        // { value: '0x44d', text: 'ZKEVM' },
      ])
      const currentAccount = ref(null);
      const isConnected = ref(false);
      const tab = ref('biconomy');
      let USDCPolygon;
      let USDCGnosis;
      let cUSDCelo;
      let tokenContract;
      const bridgeAddressPolygon = process.env.VUE_APP_BRIDGE_ADDRESS_POLYGON_SAFE
      const bridgeAddressGnosis = process.env.VUE_APP_BRIDGE_ADDRESS_GNOSIS_SAFE
      const bridgeAddressCelo = process.env.VUE_APP_BRIDGE_ADDRESS_CELO_SAFE
      let destinationAddress;
      let transactionLink = ref('')
      let tokenBalance = ref(0)
      let balanceError = ref('')
      let transactionStatus = ref(null);
      let transactionPending = ref(false);

      const getTokenBalance = async () => {
        const balance = await tokenContract.balanceOf(currentAccount.value);
        return balance
      }

      const getDecimal = async() =>{
        console.log('selectedTokenFrom.value', selectedTokenFrom.value) 
        if (selectedTokenFrom.value == 'USDC'){
          return 6
        }
        else if (selectedTokenFrom.value == 'cUSD'){
          return 18
        }
        else if  (selectedTokenFrom.value == 'USDT'){
          return 6
        }
        else{
          return 1
        }
      }

      const makeRawAmount = async () => {
        const decimal = await getDecimal()
        return amount.value * Math.pow(10, decimal)
      }

      const closeWindow = async() =>{
        transactionStatus.value = null;
      }
  
      const enableEthereum = async () => {
        try {
          const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' })
          if (accounts[0] !== currentAccount.value ) {
            currentAccount.value = accounts[0];
            isConnected.value = true;
            tokenBalance.value = await setChainId()
            filtredOptionsTokensFrom.value = await filterTokensFrom()
          }
        } catch (err) {
          if (err.code === 4001) {
            console.log('Please connect to MetaMask.');
          } else {
            console.error(err);
          }
        }
      };

      const selectDestAddres = async() =>{
        if (selectedChainTo.value == '0x89'){
          return bridgeAddressPolygon
        }
        else if (selectedChainTo.value == '0x64'){
          return bridgeAddressGnosis
        }
        else if (selectedChainTo.value == '0xa4ec'){
          return bridgeAddressCelo
        }
        else{
          return 0
        }
      }

      const sendTransaction = async() =>{
        if (typeof window.ethereum !== 'undefined') {
          const amountRaw = await makeRawAmount(amount.value)
          if (amountRaw > tokenBalance.value * Math.pow(10, getDecimal())){
            balanceError.value = 'Infficient balance'
            return 0
          }
          else{
            balanceError.value = ''
          }
          destinationAddress = await selectDestAddres()
          try {
              transactionPending.value = true;
              transactionStatus.value = 'Waiting for transaction...';

              const transactionResponse = await tokenContract.transfer(destinationAddress, amountRaw, {
                gasLimit: ethers.utils.hexlify(200000)
              });

              const receipt = await transactionResponse.wait();
              let preLink = ''
              if (selectedChainFrom.value == '0x89'){
                preLink = 'https://polygonscan.com/tx/'
              }
              else if (selectedChainFrom.value == '0x64'){
                preLink = 'https://gnosisscan.io/tx/'
              }
              else if (selectedChainFrom.value == '0xa4ec'){
                preLink = 'https://celoscan.io/tx/'
              }
              transactionLink.value = preLink + receipt.transactionHash
              transactionPending.value = false;

              if(receipt.status === 1) {
                  transactionStatus.value = 'Transaction complete';
                  console.log(receipt)
                  setTimeout(() => {
                    transactionStatus.value = null;
                  }, 5000);
              }
              else {
                  transactionStatus.value = 'Transaction failed';
                  setTimeout(() => {
                    transactionStatus.value = null;
                  }, 5000);
                  console.log('Transaction failed');
              }
          } catch (error) {
              transactionPending.value = false;
              transactionStatus.value = 'Transaction failed';
              console.error('Error occurred: ', error);
          }
          } else {
              console.log('Please install MetaMask!');
          }
      }

      const filterTokensFrom = async() => {
        let new_list = []
        for (let option of optionsTokens.value){
          if (option.chains.includes(selectedChainFrom.value)){
            new_list.push(option)
          }
        }
        return new_list
      }

      const setChainId = async() =>{
        let chainIdHex = await window.ethereum.request({ method: 'eth_chainId' });
        selectedChainFrom.value = chainIdHex
        selectedChainTo.value = '0x64'
        if (selectedChainFrom.value == '0x89'){
          tokenContract = USDCPolygon
        }
        else if(selectedChainFrom.value == '0x64'){
          tokenContract = USDCGnosis
        }
        else if(selectedChainFrom.value == '0xa4ec'){
          tokenContract = cUSDCelo
        }
        filtredOptionsTokensFrom.value = await filterTokensFrom()
        const balance = await getTokenBalance()
        const decimal = await getDecimal()
        return balance/Math.pow(10, decimal)
      }

      const changeFromChain = async() =>{
        let chainIdHex = await window.ethereum.request({ method: 'eth_chainId' });
        if (selectedChainFrom.value != parseInt(chainIdHex, 16)){
          chainIdHex = '0x' + parseInt(selectedChainFrom.value).toString(16)
          await window.ethereum.request({
            method: 'wallet_switchEthereumChain',
            params: [{ chainId: chainIdHex }], //
          });
          return 0
        }
       
        if (selectedChainFrom.value == '0x89'){
          tokenContract = USDCPolygon
          selectedTokenFrom.value = 'USDC'
        }
        else if(selectedChainFrom.value == '0x64'){
          tokenContract = USDCGnosis
          selectedTokenFrom.value = 'USDC'
        }
        else if(selectedChainFrom.value == '0xa4ec'){
          tokenContract = cUSDCelo
          selectedTokenFrom.value = 'cUSD'
        }
        const balance = await getTokenBalance()
        const decimal = await getDecimal()
        return balance/Math.pow(10, decimal)
      }

      const createContractUSDCPolygon = async() =>{
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const contractAddress = process.env.VUE_APP_USDC_ADDRESS_POLYGON;
        const contract = new ethers.Contract(contractAddress, abiUSDCPolygon, signer);
        return contract
      }

      const createContractUSDCGnosis = async() =>{
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const contractAddress = process.env.VUE_APP_USDC_ADDRESS_GNOSIS;
        const contract = new ethers.Contract(contractAddress, abiUSDCGnosis, signer);
        return contract
      }

      const createContractcUSDCelo = async() =>{
        const provider = new ethers.providers.Web3Provider(window.ethereum);
        const signer = provider.getSigner();
        const contractAddress = process.env.VUE_APP_CUSD_ADDRESS_CELO;
        const contract = new ethers.Contract(contractAddress, abicUSDCelo, signer);
        return contract
      }
  
      onMounted(async () => {
        const provider = await detectEthereumProvider();
        const accounts = await window.ethereum.request({ method: 'eth_accounts' });

  
        if ((accounts.length != 0)) {
          USDCPolygon = await createContractUSDCPolygon();
          USDCGnosis = await createContractUSDCGnosis();
          cUSDCelo = await createContractcUSDCelo();
          currentAccount.value = accounts[0];
          isConnected.value = true;
          if (provider !== window.ethereum) {
            console.error('Do you have multiple wallets installed?');
          } else {
            window.ethereum.on('chainChanged', () => window.location.reload());
            window.ethereum.on('accountsChanged', enableEthereum);
          }
          tokenBalance.value = await setChainId()
          console.log('tokenBalance', tokenBalance)
        } else {
          console.log('Connect metamask');
          filtredOptionsTokensFrom.value = await filterTokensFrom()
        }
      });
      watch(selectedChainFrom, () => {
        changeFromChain()
      });
  
      return {
        balanceError,
        optionsTokens,
        optionsChains,
        filtredOptionsTokensFrom,
        closeWindow,
        selectedTokenFrom,
        selectedChainFrom,
        selectedChainTo,
        amount,
        tab, 
        currentAccount,
        tokenBalance,
        enableEthereum,
        sendTransaction,
        isConnected,
        transactionStatus,
        transactionPending
      };
    }
  }
</script>
<style>
  body {
    font-family: Arial, sans-serif;
    background-color: #F0F0F0;
    padding: 20px;
}
form {
    background-color: white;
    border-radius: 5px;
    padding: 20px;
    margin-bottom: 20px;
    box-shadow: 0 2px 5px rgba(0, 0, 0, 0.15);
}
h2 {
    color: #333;
    margin-bottom: 20px;
}
button {
    background-color: #007BFF;
    color: white;
    border: none;
    padding: 10px 20px;
    border-radius: 5px;
    cursor: pointer;
    padding: 10px 20px;
}
button:hover {
    background-color: #0056b3;
}
label {
    display: block;
    margin-bottom: 10px;
}
input {
    width: 90%;
    padding: 10px;
    border-radius: 5px;
    border: 1px solid #CCC;
    margin-bottom: 20px;
}
.container {
    display: flex;
    justify-content: center;
}
.account{
    margin-bottom: 20px;
    font-size: large;
}
.background {
    width: 50%;
    background-color: lightgray;
    padding: 20px;
}
.transaction-contrainer{
  position: fixed;
  top: 150px;
  left: 50%;
  transform: translate(-50%, 0);
  width: 300px;
  height: 270px;
  background-color: white;
  z-index: 1000;
  padding: 20px;
  box-shadow: 0 0 30px rgba(0, 0, 0, 0.2);
}
.transaction-status {
  width: 300px;
  height: 200px;
  display: flex;
  justify-content: center;
  align-items: center;
  border-radius: 5px;
  text-align: center;
}
.loader {
  border: 16px solid #f3f3f3;
  border-radius: 50%;
  border-top: 16px solid #3498db;
  width: 60px;
  height: 60px;
  animation: spin 2s linear infinite;
  margin-right: 10px;
}

@keyframes spin {
  0% { transform: rotate(0deg); }
  100% { transform: rotate(360deg); }
}
.menu {
    display: flex;
    justify-content: space-between;
}

.menu button {
    width:100%;
    padding: 10px 20px;
    background-color: #c8c8d3;
    border: 1px solid #afadad;
    border-radius: 0px;
    border-top-left-radius: 5px;
    border-top-right-radius: 5px;
    cursor: pointer;
    transition: background-color 0.3s ease;
}
.form-control-threshold{
    width: 30%;
    text-align: center;
}

.menu button:hover {
    background-color: #afc5e4;
}

.menu .active {
    background-color: #007bff;
    color: white;
}
.bridge-container {
    padding: 20px;
    box-sizing: border-box;
}

.row {
    display: flex;
    justify-content: space-between;
}

.select {
    box-sizing: border-box;
}
.destination{
  text-align: left;
}
.destination-to{
  margin-top:20px;
}
.balance-error{
  color:red;
  font-size: 11px;
}
.amount{
  margin-top:20px;
}
.amount-and-balance{
  display: flex;
  justify-content: space-between;
}
.input-amount{
  padding-left: 20px;
  max-width: 100%;
  background-color: #f0f0f0;
  border-radius: 10px;
  border: solid;
  border-width: 1px;
}

.select-100 {
  text-align: center;
  padding-left: 20px;
  background-color: #f0f0f0;
  margin-right: -1px;
  border-radius: 10px; 
  border: solid;
  border-width: 1px;
  height: 40px;
  width: 320px;
}

.select-80 {
  padding-left: 20px;
  background-color: #f0f0f0;
  margin-right: -1px;
  border-radius: 10px 0px 0px 10px; 
  border: solid;
  border-width: 1px;
  height: 40px;
  width: 231px;
}

.select-20 {
  padding-left: 20px;
  background-color: #f0f0f0;
  border-radius: 0px 10px 10px 0px; 
  border: solid;
  border-width: 1px;
  height: 40px;
  width: 118px;
}
.label-address {
    cursor: pointer;
}

.top-right-buttons {
    position: absolute;
    top: 0;
    right: 0;
    display: flex;
    align-items: center;
}

.connect-button,
.disconnect-button {
    margin: 20px;
}
.container {
  display: flex;
  flex-direction: column;
  align-items: center;
}
.main-container{
  max-width: 400px;
  width: 100%;
}

.form-container {
  max-width: 100%;
}

@media (max-width: 768px) {
.form-container {
    max-width: 100%;
}
}
</style>

  