# KipuBankV2

KipuBankV2 es una evolución del contrato original KipuBank, orientada a producción. Implementa una bóveda personal multi-token con control de acceso, contabilidad interna en USD, y seguridad reforzada. Está diseñada para ser modular, segura y extensible.

---

## Mejoras Realizadas

Esta versión incorpora múltiples mejoras técnicas y estructurales:

- **Control de Acceso**: Se integró `AccessControl` de OpenZeppelin con roles `DEFAULT_ADMIN_ROLE` y `MANAGER_ROLE` para restringir funciones sensibles.
- **Soporte Multi-token**: Se agregó compatibilidad con tokens ERC-20, permitiendo depósitos y retiros en múltiples activos. ETH se representa como `address(0)`.
- **Contabilidad Interna**: Se implementaron mappings anidados para saldos por usuario y por token, junto con contadores de operaciones.
- **Eventos y Errores Personalizados**: Se definieron eventos para trazabilidad (`Deposited`, `Withdrawn`, etc.) y errores personalizados (`BankCapExceeded`, `ZeroDeposit`, etc.) para debugging eficiente.
- **Oráculo Chainlink**: Se utiliza `AggregatorV3Interface` para obtener el precio ETH/USD y controlar el límite global (`bankCapUSD`) en USD.
- **Conversión de Decimales**: Se manejan distintos decimales de tokens y se normalizan a 6 decimales (USDC) para contabilidad interna.
- **Seguridad y Eficiencia**: Se aplicaron patrones como `checks-effects-interactions`, uso de `immutable`, `constant`, `unchecked`, mutex para reentrancy, y transferencias nativas seguras.

---

## Instrucciones de Despliegue e Interacción

### Requisitos

- Solidity ^0.8.30
- Remix
- Conexión a Chainlink ETH/USD Price Feed (por ejemplo, en Sepolia)

### Despliegue en Remix

1. Copiar el archivo `KipuBankV2.sol` en la carpeta `/src`.
2. Seleccionar el compilador Solidity 0.8.30.
3. En el constructor, ingresar:
   - `withdrawalLimit`: en wei (ej. `1000000000000000000` para 1 ETH)
   - `bankCapUSD`: en 6 decimales (ej. `50000000` para $50,000 USD)
   - `priceFeed`: dirección del contrato Chainlink ETH/USD en la red seleccionada
4. (Opcional) Enviar ETH junto al despliegue para depósito inicial.

### Interacción

- `deposit()`: Deposita ETH en la bóveda personal.
- `withdraw(uint256 amount)`: Retira ETH si hay saldo suficiente y no se excede el límite.
- `depositToken(address token, uint256 amount)`: Deposita tokens ERC-20.
- `withdrawToken(address token, uint256 amount)`: Retira tokens ERC-20.
- `getVaultBalanceOf(address user)`: Consulta saldo en ETH.
- `getVaultBalanceOfToken(address user, address token)`: Consulta saldo en token.
- `convertETHtoUSD(uint256 amountWei)`: Convierte ETH a USD.
- `convertTokenToUSD(address token, uint256 amount)`: Convierte token a USD.

---

## Decisiones de Diseño y Trade-offs

- **Uso de `address(0)` para ETH**: Permite unificar la lógica de contabilidad entre ETH y tokens sin duplicar estructuras.
- **Cache de decimales**: Mejora el rendimiento en tokens con decimales estables, evitando llamadas repetidas a `decimals()`.
- **Modificadores reutilizables**: Centralizan validaciones comunes, mejorando legibilidad y seguridad.
- **No se usaron librerías externas**: Se priorizó mantener todo en un solo archivo para facilitar revisión, aunque modularizar sería ideal en producción.
- **No se incluyó lógica de precios para tokens ERC-20**: Se asumió equivalencia 1:1 con USDC para simplificar la conversión. Esto puede extenderse en futuras versiones con oráculos adicionales.

---
