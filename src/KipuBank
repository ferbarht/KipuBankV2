// SPDX-License-Identifier: MIT
pragma solidity ^0.8.30;

import "@openzeppelin/contracts/access/AccessControl.sol";
import "@openzeppelin/contracts/token/ERC20/IERC20.sol";
import "@openzeppelin/contracts/token/ERC20/extensions/IERC20Metadata.sol";
import "@chainlink/contracts/src/v0.8/interfaces/AggregatorV3Interface.sol";

/// @title KipuBankV2 - Bóveda personal multi-token con control de acceso y límite en USD vía Chainlink
/// @author Fernando
/// @notice Deposita y retira ETH o tokens ERC-20 en bóvedas personales, con roles administrativos y límite en USD
/// @dev Usa AccessControl, soporta múltiples tokens, integra oráculo Chainlink, optimiza gas con cache de decimales
contract KipuBankV2 is AccessControl {
    /*//////////////////////////////////////////////////////////////
                                 ROLES
    //////////////////////////////////////////////////////////////*/

    /// @notice Rol administrativo para funciones sensibles
    bytes32 public constant MANAGER_ROLE = keccak256("MANAGER_ROLE");

    /*//////////////////////////////////////////////////////////////
                                 CONSTANTES Y TIPOS
    //////////////////////////////////////////////////////////////*/

    /// @notice Dirección usada para representar ETH como token nativo
    address public constant NATIVE_TOKEN = address(0);

    /*//////////////////////////////////////////////////////////////
                                 ESTADO
    //////////////////////////////////////////////////////////////*/

    /// @notice Límite máximo por retiro por transacción (wei)
    /// @dev Inmutable, fijado en el constructor
    uint256 public immutable withdrawalLimit;
    
    /// @notice Límite global acumulado de depósitos permitidos en USD (6 decimales)
    /// @dev Inmutable, fijado en el constructor
    uint256 public immutable bankCapUSD;

    /// @notice Instancia del oráculo Chainlink ETH/USD
    AggregatorV3Interface public immutable priceFeed;

    /// @dev Mutex simple para protección contra reentrancy
    uint256 private _locked;

    /// @notice Total acumulado depositado en ETH (wei)
    uint256 public totalDeposited;

    /// @notice Saldo de ETH por usuario
    mapping(address => uint256) private vaults;

    /// @notice Contador de depósitos en ETH por usuario
    mapping(address => uint256) public depositCount;

    /// @notice Contador de retiros en ETH por usuario
    mapping(address => uint256) public withdrawalCount;

    /// @notice Saldo por usuario y token ERC-20
    mapping(address => mapping(address => uint256)) private vaultsByToken;

    /// @notice Contador de depósitos por usuario y token
    mapping(address => mapping(address => uint256)) public depositCountByToken;

    /// @notice Contador de retiros por usuario y token
    mapping(address => mapping(address => uint256)) public withdrawalCountByToken;

    /// @notice Total acumulado depositado por token
    mapping(address => uint256) public totalDepositedByToken;

    /// @notice Cache de decimales por token para optimizar gas
    mapping(address => uint8) public tokenDecimalsCache;

    /*//////////////////////////////////////////////////////////////
                                 EVENTOS
    //////////////////////////////////////////////////////////////*/

    /// @notice Evento emitido cuando se deposita ETH
    /// @param user Dirección del depositante
    /// @param amount Monto depositado en wei
    event Deposited(address indexed user, uint256 amount);

    /// @notice Evento emitido cuando se retira ETH
    /// @param user Dirección del que retira
    /// @param amount Monto retirado en wei
    event Withdrawn(address indexed user, uint256 amount);

    /// @notice Evento emitido cuando se deposita un token ERC-20
    /// @param user Dirección del depositante
    /// @param token Dirección del token
    /// @param amount Monto depositado
    event TokenDeposited(address indexed user, address indexed token, uint256 amount);

    /// @notice Evento emitido cuando se retira un token ERC-20
    /// @param user Dirección del que retira
    /// @param token Dirección del token
    /// @param amount Monto retirado
    event TokenWithdrawn(address indexed user, address indexed token, uint256 amount);

    /*//////////////////////////////////////////////////////////////
                                 ERRORES
    //////////////////////////////////////////////////////////////*/

    /// @notice Se lanza si el depósito excede el límite global en USD
    error BankCapExceeded(uint256 attemptedUSD, uint256 bankCapUSD);

    /// @notice Se lanza si el retiro excede el límite por transacción
    error WithdrawalLimitExceeded(uint256 requested, uint256 limit);

    /// @notice Se lanza si el usuario no tiene saldo suficiente
    error InsufficientVaultBalance(uint256 requested, uint256 available);

    /// @notice Se lanza si la transferencia nativa falla
    error NativeTransferFailed(address to, uint256 amount);

    /// @notice Se lanza si se intenta depositar 0
    error ZeroDeposit();

    /// @notice Se lanza si se detecta reentrancy
    error Reentrancy();

    /// @notice Se lanza si se intenta depositar ETH vía depositToken
    error UseNativeDeposit();

    /// @notice Se lanza si el token tiene decimales no soportados
    error UnsupportedDecimals();

    /*//////////////////////////////////////////////////////////////
                                MODIFICADORES
    //////////////////////////////////////////////////////////////*/

    ///@notice Verifica que el depósito en ETH no sea cero
    ///dev Requiere que msg.value > 0, de lo contrario revierte con ZeroDeposit
    modifier nonZeroDeposit() {
        if (msg.value == 0) revert ZeroDeposit();
        _;
    }

    ///@notice Previene reentrancy usando un mutex simple
    ///@dev Requiere que _locked == 0, lo bloquea durante la ejecución
    modifier nonReentrant() {
        if (_locked == 1) revert Reentrancy();
        _locked = 1;
        _;
        _locked = 0;
    }

    ///@notice Verifica que el monto a retirar no exceda el límite permitido
    ///@param amount Monto solicitado para retiro en wei
    ///@dev Compara contra withdrawalLimit y revierte si se excede
    modifier validWithdrawal(uint256 amount) {
        if (amount > withdrawalLimit) revert WithdrawalLimitExceeded(amount, withdrawalLimit);
        _;
    }

    ///@notice Verifica que el usuario tenga saldo suficiente en ETH
    ///@param amount Monto solicitado para retiro en wei
    ///@dev Compara contra vaults[msg.sender] y revierte si es insuficiente
    modifier hasSufficientBalance(uint256 amount) {
        uint256 balance = vaults[msg.sender];
        if (amount > balance) revert InsufficientVaultBalance(amount, balance);
        _;
    }

    ///@notice Verifica que el depósito en ETH no exceda el límite global en USD
    ///@param amountWei Monto a depositar en wei
    ///@dev Convierte a USD usando Chainlink y compara contra bankCapUSD
    modifier validDepositCapUSD(uint256 amountWei) {
        uint256 newTotalWei = totalDeposited + amountWei;
        uint256 newTotalUSD = convertETHtoUSD(newTotalWei);
        if (newTotalUSD > bankCapUSD) revert BankCapExceeded(newTotalUSD, bankCapUSD);
        _;
    }

    ///@notice Verifica que el depósito en token ERC-20 no exceda el límite global en USD
    ///@param token Dirección del token ERC-20
    ///@param amount Monto a depositar en unidades del token
    ///@dev Convierte a USD y compara contra bankCapUSD
    modifier validTokenDepositCapUSD(address token, uint256 amount) {
        uint256 newTotal = totalDepositedByToken[token] + amount;
        uint256 newTotalUSD = convertTokenToUSD(token, newTotal);
        if (newTotalUSD > bankCapUSD) revert BankCapExceeded(newTotalUSD, bankCapUSD);
        _;
    }

    /*//////////////////////////////////////////////////////////////
                               CONSTRUCTOR
    //////////////////////////////////////////////////////////////*/

    ///@notice Inicializa el contrato con límites y oráculo
    ///@param _withdrawalLimit Límite por retiro en wei
    ///@param _bankCapUSD Límite global de depósitos en USD (6 decimales)
    ///@param _priceFeed Dirección del oráculo Chainlink ETH/USD
    ///@dev Asigna roles, configura límites y permite depósito inicial opcional
    constructor(
        uint256 _withdrawalLimit,
        uint256 _bankCapUSD,
        address _priceFeed
    ) payable {
        withdrawalLimit = _withdrawalLimit;
        bankCapUSD = _bankCapUSD;
        priceFeed = AggregatorV3Interface(_priceFeed);
        _locked = 0;

        _grantRole(DEFAULT_ADMIN_ROLE, msg.sender);
        _grantRole(MANAGER_ROLE, msg.sender);

        if (msg.value > 0) {
            _handleDeposit(msg.sender, msg.value);
        }
    }

    /*//////////////////////////////////////////////////////////////
                             FUNCIONES EXTERNAS
    //////////////////////////////////////////////////////////////*/

    ///@notice Deposita ETH en la bóveda personal del remitente
    ///@dev Valida que el monto no sea cero y que no exceda el límite global en USD
    function deposit()
        external
        payable
        nonZeroDeposit
        nonReentrant
        validDepositCapUSD(msg.value)
    {
        _handleDeposit(msg.sender, msg.value);
    }

    ///@notice Retira ETH de la bóveda personal del remitente
    ///@param amount Monto a retirar en wei
    ///@dev Valida límite por transacción, saldo suficiente y previene reentrancy
    function withdraw(uint256 amount)
        external
        nonReentrant
        validWithdrawal(amount)
        hasSufficientBalance(amount)
    {
        unchecked {
            vaults[msg.sender] -= amount;
            totalDeposited -= amount;
        }
        withdrawalCount[msg.sender] += 1;
        _safeTransfer(msg.sender, amount);
        emit Withdrawn(msg.sender, amount);
    }

    ///@notice Deposita tokens ERC-20 en la bóveda personal
    ///@param token Dirección del token ERC-20
    ///@param amount Monto a depositar
    ///@dev Valida que no sea ETH, que el monto no sea cero y que no exceda el límite global en USD
    function depositToken(address token, uint256 amount)
        external
        nonReentrant
        validTokenDepositCapUSD(token, amount)
    {
        if (token == NATIVE_TOKEN) revert UseNativeDeposit();
        if (amount == 0) revert ZeroDeposit();

        IERC20(token).transferFrom(msg.sender, address(this), amount);

        unchecked {
            vaultsByToken[msg.sender][token] += amount;
            totalDepositedByToken[token] += amount;
        }
        depositCountByToken[msg.sender][token] += 1;
        emit TokenDeposited(msg.sender, token, amount);
    }

    ///@notice Retira tokens ERC-20 de la bóveda personal
    ///@param token Dirección del token ERC-20
    ///@param amount Monto a retirar
    ///@dev Valida saldo suficiente y previene reentrancy
    function withdrawToken(address token, uint256 amount)
        external
        nonReentrant
    {
        uint256 balance = vaultsByToken[msg.sender][token];
        if (amount > balance) revert InsufficientVaultBalance(amount, balance);

        unchecked {
            vaultsByToken[msg.sender][token] -= amount;
            totalDepositedByToken[token] -= amount;
        }
        withdrawalCountByToken[msg.sender][token] += 1;
        IERC20(token).transfer(msg.sender, amount);
        emit TokenWithdrawn(msg.sender, token, amount);
    }

    /*//////////////////////////////////////////////////////////////
                             FUNCIONES DE VISTA
    //////////////////////////////////////////////////////////////*/

    ///@notice Devuelve el saldo de ETH del remitente
    ///@return balance Saldo disponible en wei
    function getMyVaultBalance() external view returns (uint256) {
        return vaults[msg.sender];
    }

    ///@notice Devuelve el saldo de ETH de una dirección dada
    ///@param user Dirección del usuario a consultar
    ///@return balance Saldo disponible en wei
    function getVaultBalanceOf(address user) external view returns (uint256) {
        return vaults[user];
    }

    ///@notice Devuelve el saldo de un token ERC-20 para un usuario
    ///@param user Dirección del usuario
    ///@param token Dirección del token ERC-20
    ///@return balance Saldo disponible en unidades del token
    function getVaultBalanceOfToken(address user, address token) external view returns (uint256) {
        return vaultsByToken[user][token];
    }

    ///@notice Obtiene el precio actual de ETH en USD desde Chainlink
    ///@return ethPrice Precio de ETH en USD con 8 decimales
    function getLatestETHPrice() public view returns (uint256 ethPrice) {
        (, int256 price,,,) = priceFeed.latestRoundData();
        require(price > 0, "Invalid price");
        return uint256(price); // 8 decimales
    }

    ///@notice Convierte un monto en ETH (wei) a USD (6 decimales)
    ///@param ethAmountWei Monto en wei
    ///@return usdAmount Monto equivalente en USD
    function convertETHtoUSD(uint256 ethAmountWei) public view returns (uint256 usdAmount) {
        uint256 ethPrice = getLatestETHPrice(); // 8 decimales
        uint256 ethAmountUSD = (ethAmountWei * ethPrice) / 1e18; // resultado en 8 decimales
        return ethAmountUSD / 1e2; // normalizamos a 6 decimales (USDC)
    }

    ///@notice Convierte un monto en token ERC-20 a USD (6 decimales)
    ///@param token Dirección del token ERC-20
    ///@param amount Monto en unidades del token
    ///@return usdAmount Monto equivalente en USD
    function convertTokenToUSD(address token, uint256 amount) public view returns (uint256 usdAmount) {
        if (token == NATIVE_TOKEN) {
            return convertETHtoUSD(amount);
        }

        uint8 tokenDecimals = tokenDecimalsCache[token];
        if (tokenDecimals == 0) {
            try IERC20Metadata(token).decimals() returns (uint8 dec) {
                tokenDecimals = dec;
            } catch {
                revert UnsupportedDecimals();
            }
        }

        if (tokenDecimals > 18) revert UnsupportedDecimals();

        uint256 scaled = amount * (10 ** (18 - tokenDecimals));
        return scaled / 1e12; // normalizado a 6 decimales
    }

    /*//////////////////////////////////////////////////////////////
                            FUNCIONES INTERNAS
    //////////////////////////////////////////////////////////////*/

    ///@notice Maneja la lógica de depósito en ETH
    ///@param sender Dirección del depositante
    ///@param amount Monto depositado en wei
    function _handleDeposit(address sender, uint256 amount) internal {
        unchecked {
            vaults[sender] += amount;
            totalDeposited += amount;
        }
        depositCount[sender] += 1;
        emit Deposited(sender, amount);
    }

    ///@notice Realiza transferencia nativa segura de ETH
    ///@param to Dirección receptora
    ///@param amount Monto en wei a transferir
    function _safeTransfer(address to, uint256 amount) private {
        (bool ok, ) = to.call{value: amount}("");
        if (!ok) revert NativeTransferFailed(to, amount);
    }

    /*//////////////////////////////////////////////////////////////
                          RECEIVE / FALLBACK
    //////////////////////////////////////////////////////////////*/

    ///@notice Recibe ETH directo y lo trata como deposit()
    receive()
        external
        payable
        nonZeroDeposit
        nonReentrant
        validDepositCapUSD(msg.value)
    {
        _handleDeposit(msg.sender, msg.value);
    }

    ///@notice Fallback: acepta datos y ETH, si llega ETH se comporta como deposit()
    fallback()
        external
        payable
        nonZeroDeposit
        nonReentrant
        validDepositCapUSD(msg.value)
    {
        _handleDeposit(msg.sender, msg.value);
    }
}
