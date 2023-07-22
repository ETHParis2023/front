<template>
  <div class="container">
    <div class="top-right-buttons">
      <button v-if="!isConnected" id="connectWallet" class="connect-button" @click="enableEthereum">Connect wallet</button>
      <button v-else id="disconnectWallet" class="disconnect-button">{{ currentAccount.slice(0, 6)+'......'+currentAccount.slice(-6) }}</button>
    </div>
    <div class="account">
      <h2 class="mb-3">Account Abstraction Bridge: <span id="currentAccount"></span></h2>
    </div>
    <div class="main-container">
      <div class="form-container">
        <form id="sendTransaction" class="mb-5" @submit.prevent="sendTransaction">
          <h2 class="mb-3">Biconomy+Safe</h2>
          <div class="bridge-container">
            <div class="destination-from">
              <div class="destination">
                From:
              </div>
              <div class="row">
                <select class="select select-80" v-model="selectedTokenFrom">
                  <option v-for="option in optionsTokens" :key="option.value" :value="option.value">
                    {{ option.text }}
                  </option>
                </select>
                <select class="select select-20" v-model="selectedChainFrom">
                  <option v-for="option in optionsChains" :key="option.value" :value="option.value">
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
                <select class="select select-80" v-model="selectedTokenTo">
                  <option v-for="option in optionsTokens" :key="option.value" :value="option.value">
                    {{ option.text }}
                  </option>
                </select>
                <select class="select select-20" v-model="selectedChainTo">
                  <option v-for="option in optionsChains" :key="option.value" :value="option.value">
                    {{ option.text }}
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
            <button :disabled="!currentAccount" type="submit" class="btn btn-primary">Transfer</button>
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
  
  export default {
    setup() {
      let selectedTokenFrom = ref('USDC')
      let selectedChainFrom = ref('0x89')
      let selectedTokenTo = ref('USDC')
      let selectedChainTo = ref('0x64')
      let amount = ref()
      
      const optionsTokens = ref([
        { value: 'USDC', text: 'USDC' },
      ])
      const optionsChains = ref([
        { value: '0x89', text: 'Polygon' },
        { value: '0x64', text: 'Gnosis' },
        { value: '0x25', text: 'Celo' },
      ])
      const currentAccount = ref(null);
      const isConnected = ref(false);
      const tab = ref('biconomy');
      let USDCPolygon;
      let USDCGnosis;
      let USDCContract;
      const bridgeAddressPolygonSafe = process.env.VUE_APP_BRIDGE_ADDRESS_POLYGON_SAFE
      const bridgeAddressGnosisSafe = process.env.VUE_APP_BRIDGE_ADDRESS_GNOSIS_SAFE
      let bridgeAddressSafe;
      const decimal = 6;
      let tokenBalance = ref(0)
      let balanceError = ref('')

      const getTokenBalance = async () => {
        const balance = await USDCContract.balanceOf(currentAccount.value);
        return balance
      }

      const makeRawAmount = async () => {
        return amount.value * Math.pow(10, decimal)
      }

  
      const enableEthereum = async () => {
        try {
          const accounts = await window.ethereum.request({ method: 'eth_requestAccounts' })
          if (accounts[0] !== currentAccount.value ) {
            currentAccount.value = accounts[0];
            isConnected.value = true;
          }
        } catch (err) {
          if (err.code === 4001) {
            console.log('Please connect to MetaMask.');
          } else {
            console.error(err);
          }
        }
      };

      const sendTransaction = async() =>{
        if (typeof window.ethereum !== 'undefined') {
          const amountRaw = await makeRawAmount(amount.value)
          if (amountRaw > tokenBalance.value * Math.pow(10, decimal)){
            balanceError.value = 'Infficient balance'
            return 0
          }
          else{
            balanceError.value = ''
          }
          try {
              const transactionResponse = await USDCContract.transfer(bridgeAddressSafe, amountRaw, {
              gasLimit: ethers.utils.hexlify(200000)
            });

            const receipt = await transactionResponse.wait();

            if(receipt.status === 1) {
                console.log(receipt)
            }
            else {
                console.log('Transaction failed');
            }
          } catch (error) {
              console.error('Error occurred: ', error);
          }
          } else {
              console.log('Please install MetaMask!');
          }
      }

      const setChainId = async() =>{
        let chainIdHex = await window.ethereum.request({ method: 'eth_chainId' });
        selectedChainFrom.value = chainIdHex
        selectedChainTo.value = '0x25'
        if (selectedChainFrom.value == '0x89'){
          USDCContract = USDCPolygon
          bridgeAddressSafe = bridgeAddressPolygonSafe
        }
        else if(selectedChainFrom.value == '0x64'){
          USDCContract = USDCGnosis
          bridgeAddressSafe = bridgeAddressGnosisSafe
        }
        const balance = await getTokenBalance()
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
        }
       
        if (selectedChainFrom.value == '0x89'){
          USDCContract = USDCPolygon
          bridgeAddressSafe = bridgeAddressPolygonSafe
        }
        else if(selectedChainFrom.value == '0x64'){
          USDCContract = USDCGnosis
          bridgeAddressSafe = bridgeAddressGnosisSafe
        }
        const balance = await getTokenBalance()
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
  
      onMounted(async () => {
        const provider = await detectEthereumProvider();

        USDCPolygon = await createContractUSDCPolygon();
        USDCGnosis = await createContractUSDCGnosis();
  
        if (provider) {
          await enableEthereum()
          if (provider !== window.ethereum) {
            console.error('Do you have multiple wallets installed?');
          } else {
            window.ethereum.on('chainChanged', () => window.location.reload());
            window.ethereum.on('accountsChanged', enableEthereum);
          }
        } else {
          console.log('Please install MetaMask!');
        }
        tokenBalance.value = await setChainId()
      });
      watch(selectedChainFrom, () => {
        changeFromChain()
      });
  
      return {
        balanceError,
        optionsTokens,
        optionsChains,
        selectedTokenFrom,
        selectedChainFrom,
        selectedTokenTo,
        selectedChainTo,
        amount,
        tab, 
        currentAccount,
        tokenBalance,
        enableEthereum,
        sendTransaction,
        isConnected
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
  width: 300px;
  background-color: #f0f0f0;
  border-radius: 10px;
  border: solid;
  border-width: 1px;
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

  