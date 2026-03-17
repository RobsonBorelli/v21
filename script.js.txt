// ========================================
// CONFIGURAÇÕES DA REDE BASE SEPOLIA E TOKENS
// ========================================
const CONFIG = {
  presaleEndDate: new Date('2026-03-18T20:00:00Z').getTime(),
  exchangeRate: 10000,
  tokenPrice: 0.0001,
  network: {
    name: 'Base Sepolia',
    chainId: '0x14a33',
    chainIdDecimal: 84532,
    rpcUrl: 'https://sepolia.base.org',
    blockExplorer: 'https://sepolia.basescan.org'
  },
  tokens: {
    USDC: {
      address: '0x036CbD53842c5426634e7929541eC2318f3dCF7e',
      decimals: 6,
      symbol: 'USDC'
    },
    USDT: {
      address: '0x50c5725949A6F0c72E6C4a641F24049A917DB0Cb',
      decimals: 6,
      symbol: 'USDT'
    }
  },
  sfomoToken: {
    address: '0x8cc88849C4851e185A8FfF2C554A58FFe1aC6b68',
    decimals: 18,
    symbol: 'SFOMO'
  },
  presaleContract: '0x1366251D9650fd3987c0De313C04b1C73f03C83d'
};

// ========================================
// ABIs MÍNIMAS NECESSÁRIAS
// ========================================
const ERC20_ABI = [
  {
    "constant": true,
    "inputs": [{ "name": "_owner", "type": "address" }],
    "name": "balanceOf",
    "outputs": [{ "name": "balance", "type": "uint256" }],
    "type": "function"
  },
  {
    "constant": false,
    "inputs": [
      { "name": "_spender", "type": "address" },
      { "name": "_value", "type": "uint256" }
    ],
    "name": "approve",
    "outputs": [{ "name": "", "type": "bool" }],
    "type": "function"
  },
  {
    "constant": true,
    "inputs": [
      { "name": "_owner", "type": "address" },
      { "name": "_spender", "type": "address" }
    ],
    "name": "allowance",
    "outputs": [{ "name": "", "type": "uint256" }],
    "type": "function"
  }
];

const PRESALE_ABI = [
  {
    "inputs": [{ "name": "amount", "type": "uint256" }],
    "name": "buyTokens",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "tokenPrice",
    "outputs": [{ "name": "", "type": "uint256" }],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [{ "name": "user", "type": "address" }],
    "name": "claimableAmount",
    "outputs": [{ "name": "", "type": "uint256" }],
    "stateMutability": "view",
    "type": "function"
  },
  {
    "inputs": [],
    "name": "claimTokens",
    "outputs": [],
    "stateMutability": "nonpayable",
    "type": "function"
  }
];

// ========================================
// ESTADO DA APLICAÇÃO
// ========================================
let web3 = null;
let account = null;
let currentToken = 'USDC';
let isApproved = false;
let isConnecting = false;

// ========================================
// ELEMENTOS DO DOM
// ========================================
const elements = {};

// ========================================
// INICIALIZAÇÃO (SEM AUTO-CONEXÃO)
// ========================================
document.addEventListener('DOMContentLoaded', () => {
  console.log('🦸 SFOMO Token - Pre-Sale Active!');
  console.log(`🌐 Network: ${CONFIG.network.name} (Chain ID: ${CONFIG.network.chainIdDecimal})`);
  
  cacheElements();
  setupEventListeners();
  initializeApp();
  startPresaleCountdown();
  
  console.log('✅ Ready! Click "Connect Wallet" to start.');
});

// ========================================
// CACHE DE ELEMENTOS
// ========================================
function cacheElements() {
  elements.payAmount = document.getElementById('payAmount');
  elements.receiveAmount = document.getElementById('receiveAmount');
  elements.payToken = document.getElementById('payToken');
  elements.payTokenLabel = document.getElementById('payTokenLabel');
  elements.connectWalletBtn = document.getElementById('connectWalletBtn');
  elements.approveTokenBtn = document.getElementById('approveTokenBtn');
  elements.confirmPurchaseBtn = document.getElementById('confirmPurchaseBtn');
  elements.disconnectWalletBtn = document.getElementById('disconnectWalletBtn');
  elements.walletInfo = document.getElementById('walletInfo');
  elements.walletAddress = document.getElementById('walletAddress');
  elements.walletBalance = document.getElementById('walletBalance');
  elements.contractAddress = document.getElementById('contractAddress');
  elements.usdcBalance = document.getElementById('usdcBalance');
  elements.usdtBalance = document.getElementById('usdtBalance');
  elements.approveTokenName = document.getElementById('approveTokenName');
  elements.walletStatus = document.getElementById('walletStatus');
  elements.statusAddress = document.getElementById('statusAddress');
  elements.statusBalance = document.getElementById('statusBalance');
  elements.approvalText = document.getElementById('approvalText');
  elements.currentRate = document.getElementById('currentRate');
  elements.claimSection = document.getElementById('claimSection');
  elements.claimBtn = document.getElementById('claimBtn');
  elements.claimableAmount = document.getElementById('claimableAmount');
  elements.claimStatus = document.getElementById('claimStatus');
  elements.presaleCountdown = document.getElementById('presaleCountdown');
  elements.daysLeft = document.getElementById('daysLeft');
  elements.hoursLeft = document.getElementById('hoursLeft');
  elements.minutesLeft = document.getElementById('minutesLeft');
  elements.secondsLeft = document.getElementById('secondsLeft');
}

// ========================================
// INICIALIZAÇÃO APP
// ========================================
function initializeApp() {
  calculateSwap();
  updateRateDisplay();
}

// ========================================
// EVENT LISTENERS
// ========================================
function setupEventListeners() {
  if (elements.payAmount) {
    elements.payAmount.addEventListener('input', calculateSwap);
    elements.payAmount.addEventListener('keydown', (e) => {
      if (e.key === '-') e.preventDefault();
    });
  }
  
  if (elements.payToken) {
    elements.payToken.addEventListener('change', handleTokenChange);
  }
  
  if (elements.connectWalletBtn) {
    elements.connectWalletBtn.addEventListener('click', connectWallet);
  }
  
  if (elements.approveTokenBtn) {
    elements.approveTokenBtn.addEventListener('click', approveToken);
  }
  
  if (elements.confirmPurchaseBtn) {
    elements.confirmPurchaseBtn.addEventListener('click', buyTokens);
  }
  
  if (elements.disconnectWalletBtn) {
    elements.disconnectWalletBtn.addEventListener('click', disconnectWallet);
  }
  
  if (elements.claimBtn) {
    elements.claimBtn.addEventListener('click', claimTokens);
  }
  
  if (window.ethereum) {
    window.ethereum.on('accountsChanged', handleAccountsChanged);
    window.ethereum.on('chainChanged', () => window.location.reload());
  }
}

// ========================================
// CÁLCULO DE SWAP
// ========================================
function calculateSwap() {
  const amount = parseFloat(elements.payAmount?.value) || 0;
  const received = amount * CONFIG.exchangeRate;
  if (elements.receiveAmount) {
    elements.receiveAmount.value = received.toLocaleString('en-US', {
      minimumFractionDigits: 0,
      maximumFractionDigits: 0
    });
  }
}

function updateRateDisplay() {
  if (elements.currentRate) {
    elements.currentRate.textContent = `1 ${currentToken} = ${CONFIG.exchangeRate.toLocaleString()} SFOMO`;
  }
}

function handleTokenChange() {
  currentToken = elements.payToken.value;
  if (elements.payTokenLabel) elements.payTokenLabel.textContent = currentToken;
  if (elements.approveTokenName) elements.approveTokenName.textContent = currentToken;
  updateRateDisplay();
  calculateSwap();
  if (account) {
    checkApproval();
  }
}

// ========================================
// CONEXÃO WALLET (CORRIGIDA - SEM LOOP)
// ========================================
async function connectWallet() {
  if (isConnecting) {
    console.log('⚠️ Já está conectando...');
    return;
  }
  
  console.log('🔌 Iniciando conexão...');
  
  if (!window.ethereum) {
    showNotification('MetaMask not found! Please install it.', 'error');
    return;
  }
  
  isConnecting = true;
  
  try {
    if (elements.connectWalletBtn) {
      elements.connectWalletBtn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Connecting...';
      elements.connectWalletBtn.disabled = true;
    }
    
    const accounts = await window.ethereum.request({ 
      method: 'eth_requestAccounts' 
    });
    
    if (!accounts || accounts.length === 0) {
      throw new Error('No accounts returned');
    }
    
    account = accounts[0].toLowerCase();
    console.log('✅ Conta conectada:', account);
    
    web3 = new Web3(window.ethereum);
    
    const chainId = await web3.eth.getChainId();
    console.log('🔗 Chain ID atual:', chainId);
    
    if (chainId !== CONFIG.network.chainIdDecimal) {
      console.log('⚠️ Rede errada. Trocando para Base Sepolia...');
      showNotification('Switching to Base Sepolia Testnet...', 'info');
      await switchToBaseNetwork();
      isConnecting = false;
      return;
    }
    
    onWalletConnected();
    
  } catch (error) {
    console.error('❌ Erro na conexão:', error);
    
    if (error.code === 4001) {
      showNotification('Connection rejected by user.', 'error');
    } else {
      showNotification('Connection failed: ' + error.message, 'error');
    }
    
  } finally {
    isConnecting = false;
    if (elements.connectWalletBtn) {
      elements.connectWalletBtn.innerHTML = '<i class="fas fa-wallet"></i> Connect Wallet';
      elements.connectWalletBtn.disabled = false;
    }
  }
}

// ========================================
// QUANDO WALLET CONECTA
// ========================================
function onWalletConnected() {
  console.log('✅ Wallet conectada com sucesso!');
  
  updateWalletUI();
  
  if (elements.connectWalletBtn) elements.connectWalletBtn.style.display = 'none';
  if (elements.walletInfo) elements.walletInfo.style.display = 'block';
  if (elements.walletStatus) elements.walletStatus.style.display = 'block';
  if (elements.disconnectWalletBtn) elements.disconnectWalletBtn.style.display = 'inline-flex';
  
  checkApproval();
  
  showNotification('Wallet Connected Successfully!', 'success');
  
  isConnecting = false;
  if (elements.connectWalletBtn) {
    elements.connectWalletBtn.innerHTML = '<i class="fas fa-wallet"></i> Connect Wallet';
    elements.connectWalletBtn.disabled = false;
  }
}

// ========================================
// ATUALIZA UI DA WALLET
// ========================================
async function updateWalletUI() {
  if (!account) return;
  
  const shortAddress = account.slice(0, 6) + '...' + account.slice(-4);
  
  if (elements.walletAddress) elements.walletAddress.textContent = shortAddress;
  if (elements.statusAddress) elements.statusAddress.textContent = shortAddress;
  if (elements.contractAddress) {
    elements.contractAddress.textContent = CONFIG.sfomoToken.address.slice(0, 10) + '...';
  }
  
  await updateBalances();
}

// ========================================
// ATUALIZA BALANÇOS
// ========================================
async function updateBalances() {
  if (!web3 || !account) return;
  
  try {
    const ethBalance = await web3.eth.getBalance(account);
    const ethFormatted = web3.utils.fromWei(ethBalance, 'ether');
    if (elements.walletBalance) {
      elements.walletBalance.textContent = `${parseFloat(ethFormatted).toFixed(4)} ETH`;
    }
    
    const usdcContract = new web3.eth.Contract(ERC20_ABI, CONFIG.tokens.USDC.address);
    const usdcBalance = await usdcContract.methods.balanceOf(account).call();
    const usdcFormatted = parseInt(usdcBalance) / Math.pow(10, CONFIG.tokens.USDC.decimals);
    if (elements.usdcBalance) {
      elements.usdcBalance.textContent = usdcFormatted.toFixed(2);
    }
    
    const usdtContract = new web3.eth.Contract(ERC20_ABI, CONFIG.tokens.USDT.address);
    const usdtBalance = await usdtContract.methods.balanceOf(account).call();
    const usdtFormatted = parseInt(usdtBalance) / Math.pow(10, CONFIG.tokens.USDT.decimals);
    if (elements.usdtBalance) {
      elements.usdtBalance.textContent = usdtFormatted.toFixed(2);
    }
    
    const currentBalance = currentToken === 'USDC' ? usdcFormatted : usdtFormatted;
    if (elements.statusBalance) {
      elements.statusBalance.textContent = `${currentBalance.toFixed(2)} ${currentToken}`;
    }
    
  } catch (error) {
    console.error('Erro ao atualizar balances:', error);
  }
}

// ========================================
// VERIFICA APROVAÇÃO
// ========================================
async function checkApproval() {
  if (!web3 || !account) return;
  
  try {
    const tokenConfig = CONFIG.tokens[currentToken];
    const tokenContract = new web3.eth.Contract(ERC20_ABI, tokenConfig.address);
    
    const allowance = await tokenContract.methods.allowance(
      account,
      CONFIG.presaleContract
    ).call();
    
    const amount = elements.payAmount?.value || '0';
    const amountWei = web3.utils.toWei(amount, 'mwei');
    
    isApproved = BigInt(allowance) >= BigInt(amountWei);
    
    if (isApproved) {
      if (elements.approveTokenBtn) elements.approveTokenBtn.style.display = 'none';
      if (elements.confirmPurchaseBtn) elements.confirmPurchaseBtn.style.display = 'block';
      if (elements.approvalText) {
        elements.approvalText.textContent = 'Approved ✓';
        elements.approvalText.className = 'approval-approved';
      }
    } else {
      if (elements.approveTokenBtn) elements.approveTokenBtn.style.display = 'block';
      if (elements.confirmPurchaseBtn) elements.confirmPurchaseBtn.style.display = 'none';
      if (elements.approvalText) {
        elements.approvalText.textContent = 'Not Approved';
        elements.approvalText.className = 'approval-pending';
      }
    }
    
  } catch (error) {
    console.error('Erro ao verificar aprovação:', error);
  }
}

// ========================================
// APROVAR TOKEN
// ========================================
async function approveToken() {
  const amount = elements.payAmount?.value;
  
  if (!amount || amount <= 0) {
    showNotification('Please enter a valid amount first.', 'error');
    return;
  }
  
  try {
    if (elements.approveTokenBtn) {
      elements.approveTokenBtn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Approving...';
      elements.approveTokenBtn.disabled = true;
    }
    
    const tokenConfig = CONFIG.tokens[currentToken];
    const tokenContract = new web3.eth.Contract(ERC20_ABI, tokenConfig.address);
    const amountWei = web3.utils.toWei(amount, 'mwei');
    
    const gasEstimate = await tokenContract.methods.approve(
      CONFIG.presaleContract,
      amountWei
    ).estimateGas({ from: account });
    
    const tx = await tokenContract.methods.approve(
      CONFIG.presaleContract,
      amountWei
    ).send({
      from: account,
      gas: gasEstimate
    });
    
    console.log('✅ Aprovação confirmada:', tx.transactionHash);
    
    await checkApproval();
    showNotification(`${currentToken} Approved Successfully!`, 'success');
    
    if (elements.approveTokenBtn) {
      elements.approveTokenBtn.innerHTML = `<i class="fas fa-check-circle"></i> Approve ${currentToken}`;
      elements.approveTokenBtn.disabled = false;
    }
    
  } catch (error) {
    console.error('❌ Erro na aprovação:', error);
    
    if (error.code === 4001) {
      showNotification('Transaction rejected by user.', 'error');
    } else {
      showNotification('Approval failed. Try again.', 'error');
    }
    
    if (elements.approveTokenBtn) {
      elements.approveTokenBtn.innerHTML = `<i class="fas fa-check-circle"></i> Approve ${currentToken}`;
      elements.approveTokenBtn.disabled = false;
    }
  }
}

// ========================================
// COMPRAR TOKENS
// ========================================
async function buyTokens() {
  const amount = parseFloat(elements.payAmount?.value);
  
  if (!amount || amount <= 0) {
    showNotification('Please enter a valid amount.', 'error');
    return;
  }
  
  const currentBalance = parseFloat(
    currentToken === 'USDC' ? elements.usdcBalance?.textContent : elements.usdtBalance?.textContent
  );
  
  if (amount > currentBalance) {
    showNotification(`Insufficient ${currentToken} balance.`, 'error');
    return;
  }
  
  const sfomoAmount = (amount * CONFIG.exchangeRate).toLocaleString();
  
  const confirmed = confirm(
    `Confirm Purchase:\n\n` +
    `Pay: ${amount} ${currentToken}\n` +
    `Receive: ${sfomoAmount} SFOMO\n` +
    `Price: $0.0001 per SFOMO\n\n` +
    `Proceed with transaction?`
  );
  
  if (!confirmed) return;
  
  try {
    if (elements.confirmPurchaseBtn) {
      elements.confirmPurchaseBtn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Processing...';
      elements.confirmPurchaseBtn.disabled = true;
    }
    
    const tokenConfig = CONFIG.tokens[currentToken];
    const tokenContract = new web3.eth.Contract(ERC20_ABI, tokenConfig.address);
    const presaleContract = new web3.eth.Contract(PRESALE_ABI, CONFIG.presaleContract);
    const amountWei = web3.utils.toWei(amount.toString(), 'mwei');
    
    const allowance = await tokenContract.methods.allowance(account, CONFIG.presaleContract).call();
    if (BigInt(allowance) < BigInt(amountWei)) {
      showNotification('Please approve tokens first.', 'error');
      await checkApproval();
      return;
    }
    
    const gasEstimate = await presaleContract.methods.buyTokens(amountWei).estimateGas({ from: account });
    
    const tx = await presaleContract.methods.buyTokens(amountWei).send({
      from: account,
      gas: gasEstimate
    });
    
    console.log('✅ Compra confirmada:', tx.transactionHash);
    
    showNotification(
      `🎉 Success! Purchased ${sfomoAmount} SFOMO!\nTX: ${tx.transactionHash.slice(0, 20)}...`,
      'success',
      8000
    );
    
    await updateBalances();
    
    if (elements.confirmPurchaseBtn) {
      elements.confirmPurchaseBtn.innerHTML = '<i class="fas fa-exchange-alt"></i> Swap Now';
      elements.confirmPurchaseBtn.disabled = false;
    }
    
    elements.payAmount.value = '100';
    calculateSwap();
    
  } catch (error) {
    console.error('❌ Erro na compra:', error);
    
    if (error.code === 4001) {
      showNotification('Transaction rejected by user.', 'error');
    } else {
      showNotification('Transaction failed. Check console.', 'error');
    }
    
    if (elements.confirmPurchaseBtn) {
      elements.confirmPurchaseBtn.innerHTML = '<i class="fas fa-exchange-alt"></i> Swap Now';
      elements.confirmPurchaseBtn.disabled = false;
    }
  }
}

// ========================================
// TROCAR REDE PARA BASE SEPOLIA
// ========================================
async function switchToBaseNetwork() {
  try {
    await window.ethereum.request({
      method: 'wallet_switchEthereumChain',
      params: [{ chainId: CONFIG.network.chainId }],
    });
  } catch (switchError) {
    if (switchError.code === 4902) {
      try {
        await window.ethereum.request({
          method: 'wallet_addEthereumChain',
          params: [{
            chainId: CONFIG.network.chainId,
            chainName: CONFIG.network.name,
            nativeCurrency: {
              name: 'ETH',
              symbol: 'ETH',
              decimals: 18
            },
            rpcUrls: [CONFIG.network.rpcUrl],
            blockExplorerUrls: [CONFIG.network.blockExplorer]
          }]
        });
      } catch (addError) {
        showNotification('Failed to add Base Sepolia network.', 'error');
        return;
      }
    } else {
      showNotification('Failed to switch network.', 'error');
      return;
    }
  }
  
  showNotification('Network switched! Reloading...', 'success');
  
  setTimeout(() => {
    window.location.reload();
  }, 2000);
}

// ========================================
// DESCONECTAR WALLET
// ========================================
function disconnectWallet() {
  account = null;
  web3 = null;
  isApproved = false;
  isConnecting = false;
  
  if (elements.connectWalletBtn) elements.connectWalletBtn.style.display = 'block';
  if (elements.walletInfo) elements.walletInfo.style.display = 'none';
  if (elements.walletStatus) elements.walletStatus.style.display = 'none';
  if (elements.disconnectWalletBtn) elements.disconnectWalletBtn.style.display = 'none';
  if (elements.approveTokenBtn) elements.approveTokenBtn.style.display = 'none';
  if (elements.confirmPurchaseBtn) elements.confirmPurchaseBtn.style.display = 'none';
  if (elements.approvalText) {
    elements.approvalText.textContent = 'Not Approved';
    elements.approvalText.className = 'approval-pending';
  }
  
  showNotification('Wallet disconnected.', 'info');
}

// ========================================
// HANDLERS DE MUDANÇA
// ========================================
function handleAccountsChanged(accounts) {
  if (accounts.length === 0) {
    disconnectWallet();
  } else if (accounts[0].toLowerCase() !== account) {
    account = accounts[0].toLowerCase();
    web3 = new Web3(window.ethereum);
    onWalletConnected();
  }
}

// ========================================
// CLAIM TOKENS
// ========================================
async function claimTokens() {
  if (!web3 || !account) {
    showNotification('Please connect your wallet first.', 'error');
    return;
  }
  
  if (elements.claimBtn) {
    elements.claimBtn.disabled = true;
    elements.claimBtn.innerHTML = '<i class="fas fa-spinner fa-spin"></i> Claiming...';
  }
  
  if (elements.claimStatus) {
    elements.claimStatus.textContent = 'Processing claim transaction...';
    elements.claimStatus.className = 'claim-status pending';
  }
  
  try {
    const presaleContract = new web3.eth.Contract(PRESALE_ABI, CONFIG.presaleContract);
    const gasEstimate = await presaleContract.methods.claimTokens().estimateGas({ from: account });
    
    const tx = await presaleContract.methods.claimTokens().send({
      from: account,
      gas: gasEstimate
    });
    
    console.log('✅ Claim confirmado:', tx.transactionHash);
    
    if (elements.claimStatus) {
      elements.claimStatus.textContent = `✅ Success! Tokens claimed! TX: ${tx.transactionHash.slice(0, 20)}...`;
      elements.claimStatus.className = 'claim-status success';
    }
    
    if (elements.claimBtn) {
      elements.claimBtn.innerHTML = '<i class="fas fa-check-circle"></i> Tokens Claimed';
    }
    
    showNotification('🎉 Tokens claimed successfully!', 'success', 8000);
    await updateClaimableAmount();
    
  } catch (error) {
    console.error('❌ Erro no claim:', error);
    
    if (elements.claimStatus) {
      elements.claimStatus.textContent = 'Claim failed. Please try again.';
      elements.claimStatus.className = 'claim-status error';
    }
    
    if (elements.claimBtn) {
      elements.claimBtn.disabled = false;
      elements.claimBtn.innerHTML = '<i class="fas fa-download"></i> Claim Tokens';
    }
    
    if (error.code === 4001) {
      showNotification('Transaction rejected by user.', 'error');
    } else {
      showNotification('Claim failed. Check console.', 'error');
    }
  }
}

async function updateClaimableAmount() {
  if (!web3 || !account) return;
  
  try {
    const presaleContract = new web3.eth.Contract(PRESALE_ABI, CONFIG.presaleContract);
    const claimable = await presaleContract.methods.claimableAmount(account).call();
    const claimableFormatted = parseInt(claimable) / Math.pow(10, CONFIG.sfomoToken.decimals);
    
    if (elements.claimableAmount) {
      elements.claimableAmount.textContent = `${claimableFormatted.toLocaleString('en-US', { maximumFractionDigits: 2 })} SFOMO`;
    }
  } catch (error) {
    console.error('Erro ao buscar claimable amount:', error);
    if (elements.claimableAmount) {
      elements.claimableAmount.textContent = '0 SFOMO';
    }
  }
}

function showClaimSection() {
  const claimSection = document.getElementById('claimSection');
  const swapContainer = document.querySelector('.swap-container');
  if (claimSection && swapContainer) {
    swapContainer.style.display = 'none';
    claimSection.style.display = 'block';
    updateClaimableAmount();
  }
}

// ========================================
// COUNTDOWN
// ========================================
function startPresaleCountdown() {
  const countdownInterval = setInterval(() => {
    const now = new Date().getTime();
    const distance = CONFIG.presaleEndDate - now;
    
    if (distance < 0) {
      clearInterval(countdownInterval);
      if (elements.presaleCountdown) {
        elements.presaleCountdown.innerHTML = '<span style="color: var(--red-bright); font-weight: 700; font-family: \'Bangers\', cursive; font-size: 28px;">ENDED</span>';
      }
      showClaimSection();
      return;
    }
    
    const days = Math.floor(distance / (1000 * 60 * 60 * 24));
    const hours = Math.floor((distance % (1000 * 60 * 60 * 24)) / (1000 * 60 * 60));
    const minutes = Math.floor((distance % (1000 * 60 * 60)) / (1000 * 60));
    const seconds = Math.floor((distance % (1000 * 60)) / 1000);
    
    if (elements.daysLeft) elements.daysLeft.textContent = String(days).padStart(2, '0');
    if (elements.hoursLeft) elements.hoursLeft.textContent = String(hours).padStart(2, '0');
    if (elements.minutesLeft) elements.minutesLeft.textContent = String(minutes).padStart(2, '0');
    if (elements.secondsLeft) elements.secondsLeft.textContent = String(seconds).padStart(2, '0');
  }, 1000);
}

// ========================================
// NOTIFICAÇÕES
// ========================================
function showNotification(message, type = 'info', duration = 3000) {
  const existing = document.querySelector('.notification');
  if (existing) existing.remove();
  
  const notification = document.createElement('div');
  notification.className = `notification notification-${type}`;
  notification.innerHTML = `
    <i class="fas ${type === 'success' ? 'fa-check-circle' : type === 'error' ? 'fa-exclamation-circle' : 'fa-info-circle'}"></i>
    <span style="white-space: pre-line;">${message}</span>
  `;
  
  Object.assign(notification.style, {
    position: 'fixed',
    top: '100px',
    right: '20px',
    padding: '20px 30px',
    background: type === 'success' ? '#10B981' : type === 'error' ? '#EF4444' : '#3B82F6',
    color: 'white',
    borderRadius: '12px',
    boxShadow: '0 10px 30px rgba(0,0,0,0.4)',
    display: 'flex',
    alignItems: 'center',
    gap: '12px',
    fontWeight: '600',
    zIndex: '10000',
    animation: 'slideIn 0.3s ease',
    maxWidth: '450px',
    fontSize: '15px'
  });
  
  document.body.appendChild(notification);
  
  setTimeout(() => {
    notification.style.animation = 'slideOut 0.3s ease';
    setTimeout(() => notification.remove(), 300);
  }, duration);
}

// ========================================
// LOG NO CONSOLE
// ========================================
console.log('%c🦸 SFOMO PRE-SALE ACTIVE!', 'font-size: 30px; font-weight: bold; color: #FBBF24;');
console.log('%c💰 Price: $0.0001 | Network: Base Sepolia Testnet', 'font-size: 16px; color: #10B981;');