<div align="center">
    <img src="./Imagenes/Starknet.png" style="width: 300px">
    <h1>Nadai Token NaiProxy ERC20</h1>

[![StarkWare](https://img.shields.io/badge/powered_by-StarkWare-navy?style=for-the-badge&flat&logo=Starkware)](https://starkware.co/)
[![Cairo](https://img.shields.io/badge/-%F0%9F%90%AB%20%20Cairo-red?style=for-the-badge&flat&logo=Cairo)](https://www.cairo-lang.org/)
[![Protostar](https://img.shields.io/badge/-%E2%9C%A8PROTOSTAR-blue?style=for-the-badge&flat&logo=Protostar)](https://docs.swmansion.com/protostar/)
</div>

# NaiProxy V2

Esta guía será una pequeña actualización de la antigua [Guía NaiProxy](https://github.com/Nadai2010/Nadai-NaiProxy-Starknet-ERC20) a NaiProxy V2, además de como aprender actualizar el contrato una vez llege el momento con Cairo 1.0, es necesario comprender cómo funcionan los contratos actualizables para poder migrar con éxito los contratos existentes a Cairo v1.0. En esta sección, aprenderemos cómo crear contratos actualizables mediante la codificación de un token ERC20 actualizable **NAI** [NaiTokenV1](/src/NaiTokenV1.cairo). Empezaremos desde 0, primero haremos el deploy del contrato [Proxy.cairo](/src/Proxy.cairo) con una implementación de nuestro Token ERC20 **NAI**, al que luego actiualizaremos a la versión [NaiTokenV2](/src/NaiTokenV2.cairo) con un nuevo método para la quema `burn`.

En términos simples, un contrato actualizable es aquel que le permite cambiar el código/lógica subyacente de su contrato inteligente, sin alterar necesariamente el punto de entrada (dirección del contrato) de su dApp. Esto se hace separando sus contratos en un Proxy y una implementación. El Proxy sirve como punto de entrada y también contiene el almacenamiento del contrato, mientras que la Implementación contiene el código/lógica de su dApp. Para una inmersión más profunda, consulte este artículo de David Baretto [aquí](https://medium.com/starknet-edu/creating-upgradable-smart-contracts-on-starknet-12b7d9bd60c7)

Gracias al equipo de Openzeppelin, ya tenemos una buena plantilla a seguir. Primero, necesitamos copiar el [contrato de proxy](https://github.com/OpenZeppelin/cairo-contracts/blob/main/src/openzeppelin/upgrades/presets/Proxy.cairo), en nuestro repositorio. Este contrato de proxy contiene algunas funciones importantes que debemos comprender:

- El `constructor` que toma 4 parámetros: `implementation_hash` (que es el hash de clase de nuestro contrato de implementación), `selector` que es el nombre del selector de nuestra función inicializadora (1295919550572838631247819983596733806859788957403169325509326258146877103642), `datacall_len ` (argumentos del constructor del contrato de implementación) que es la longitud de nuestra llamada y `calldata` que son los argumentos del constructor del contrato de implementación. El constructor establece el hash de implementación e inicializa el contrato de implementación.

- La función `__default__` que es responsable de redirigir cualquier llamada de función cuyo selector no se encuentre en el contrato del proxy a la implementación.

- La función `__l1_default__` que se encarga de redirigir cualquier función realizada a un @l1_handler cuyo selector no se encuentra en el contrato del proxy a la implementación.

Finalmente, creamos nuestros contratos de implementación agregando funciones como `upgrade` para actualizar el hash de implementación, `setAdmin` para configurar el administrador de proxy, `getImplementationHash` para obtener el hash de clase de contrato de implementación y `getAdmin` para obtener el administrador de proxy actual.

Tenga en cuenta que el contrato de implementación nunca debe:

1. Ser desplegado como un contrato regular. En su lugar, se debe declarar el contrato de implementación (lo que crea una clase declarada que contiene su hash y abi).
2. Establecer su estado inicial con un constructor tradicional (decorado con @constructor). En su lugar, utilice un método de inicialización que invoque al constructor Proxy.

---

## Ajustes de Entorno

Antes de empezar asegurese de tener instalado [Protostar](https://github.com/software-mansion/protostar). También tiene una guía antigua [aqui](https://github.com/Nadai2010/Nadai-ERC721-Protostar-Cairo#instalaci%C3%B3n)

Debemos instalar las librerias de OpenZeppelin usando el comando

```bash
gh repo clone OpenZeppelin/cairo-contracts
```

### Ajustes Account

En esta nueva version de protostar `0.9.0` tendremos que crear un perfil que añadirermos en [protostar.tom](/protostar.toml) en el que defineremos el usuario de cuenta que pagará el fee. Usaremos los ajustes `PARA TESTNET` aunque también estan preparados para `TESTNET2`. La cuenta de ArgentX para la guía será `0x03F878C94De81906ba1A016aB0E228D361753536681a776ddA29674FfeBB3CB0` (EN SU CASO AÑADIR LA VUESTRA) para el deploy del [Proxy.cairo](/src/Proxy.cairo). Tendremos que exportar nuestra `PRIVATE KEY` de esa cuenta de argent y pasarlo a `hexa` usando [Stark-utils](https://www.stark-utils.xyz/converter).

 **Recordar que esto solo será una de las `OPCIONES` para TESTNET o TESNET2 y no usar ni compartir ninguna PRIVATE KEY NUNCA, todo ello es provisional hasta la versión de `CAIRO 1.0`. Al reiniciar el pc o terminal tendrá que volver a exportar la clave.** 
 
 Luego pasaremos el `hexa` convertido de nuestra private key para exportarla usando el siguiente comando. (SUSITITUIR 0x1234 por vuestro hexa).

```bash
export PROTOSTAR_ACCOUNT_PRIVATE_KEY=0x1234
```

---

## Class Hash Del Proxy, V1 y V2.

Ahora compilaremos los 3 contratos para obtener su `Class Hash`, de los cuales sòlo haremos el `declare` para los 3 y el `deploy` sólo para el [Proxy](/src/Proxy.cairo). Añadiremos en el archivo [Protostar.toml](/protostar.toml) los contratos que vamos a declarar y ejecutamos el siguiente comando.

```bash
protostar build
```

![Graph](/im%C3%A1genes/tomlproxy.png)
![Graph](/im%C3%A1genes/build.png)

* Ahora realizamos los mismos pasos para NaiTokenV1 y para NaiToken V2, ejecutando dos veces más el `build`, que nos creará los dos archivos `.json` restante para pasar al `declare`.

![Graph](/im%C3%A1genes/tomlv1.png)
![Graph](/im%C3%A1genes/buildv1.png)
![Graph](/im%C3%A1genes/tomlv2.png)
![Graph](/im%C3%A1genes/buildv2.png)

---

Ahora pasaremos hacer los `declare` pero vamos guardando los Class Hash que quedarán de la siguiente manera:

* Proxy = 0xeafb0413e759430def79539db681f8a4eb98cf4196fe457077d694c6aeeb82
* NaiTokenV1 = 0x459d37251c6f901a470d5e68a9ab6db594f79aaed49553975e132f56d2107b0
* NaiTokenV2 = 0x7df12f46fee13fa7279754d62ae8cf4ffc224db70bb5cd90143659ffe9d7ba7

---

## Declare 

Revisar que la cuenta esté exportada y vamos a pasar los 3 comandos para el declare de cada uno de los contratos.

1. protostar -p testnet declare ./build/Proxy.json --max-fee auto

2. protostar -p testnet declare ./build/NaiTokenV1.json --max-fee auto

3. protostar -p testnet declare ./build/NaiTokenV2.json --max-fee auto

* [Class Hash Proxy](https://testnet.starkscan.co/class/0x00eafb0413e759430def79539db681f8a4eb98cf4196fe457077d694c6aeeb82)
* [Class Hash NaiTokenV1](https://testnet.starkscan.co/class/0x0459d37251c6f901a470d5e68a9ab6db594f79aaed49553975e132f56d2107b0)
* [Class Hash NaiTokenV2](https://testnet.starkscan.co/class/0x07df12f46fee13fa7279754d62ae8cf4ffc224db70bb5cd90143659ffe9d7ba7)

![Graph](/im%C3%A1genes/declare.png)


Como vemos por ahora todos deben de coincidir con los iniciales. Ahora haremos el `deploy` del [Proxy](/src/Proxy.cairo) usando protostar, auqnue también podriamos usar el [UDC](https://testnet.starkscan.co/contract/0x041a78e741e5af2fec34b695679bc6891742439f7afb8484ecd7766661ad02bf#write-contract) por comodidad. Para ello debemos de pasar al `constructor` que toma 4 parámetros, la `implementation_hash` (que es el hash de clase de nuestro contrato de implementación), `selector` que es el nombre del selector de nuestra función inicializadora (1295919550572838631247819983596733806859788957403169325509326258146877103642), `datacall_len ` (argumentos del constructor del contrato de implementación) que es la longitud de nuestra llamada y `calldata` que son los argumentos del constructor del contrato de implementación. El constructor establece el hash de implementación e inicializa el contrato de implementación. En nuestro sólo necesitamos ajustar la dirreción de `ArgentX` para este deploy.

```bash
protostar -p testnet deploy 0x00eafb0413e759430def79539db681f8a4eb98cf4196fe457077d694c6aeeb82 --max-fee auto -i 0x0459d37251c6f901a470d5e68a9ab6db594f79aaed49553975e132f56d2107b0 0x2dd76e7ad84dbed81c314ffe5e7a7cacfb8f4836f01af4e913f275f89a3de1a 1 0x03F878C94De81906ba1A016aB0E228D361753536681a776ddA29674FfeBB3CB0
```

![Graph](/im%C3%A1genes/deploy.png)

* [Contract Proxy](https://testnet.starkscan.co/contract/0x06295ee7376800e4d929bf93639d4c65156f8666f2586084c4ad80a2a7addf50)


Si todo ha ido bien nuestro contrato de Proxy con nuestro Token **NaiV1** ha sido desplegado, revise la siguiente foto y podra comprobar que aún no tenemos los métodos de `burn` que añadiremos a continuación, iremos al [upgrade](https://testnet.starkscan.co/contract/0x06295ee7376800e4d929bf93639d4c65156f8666f2586084c4ad80a2a7addf50#write-contract) y vamos a pasar **NaiV2** (su Class Hash que antes declaramos)

![Graph](/im%C3%A1genes/v1.png)

![Graph](/im%C3%A1genes/upgra.png)

* [Hash Upgrade](https://testnet.starkscan.co/tx/0x4be535bfc0c89a4f4993217ba80fa03228de563c4cc5eb376527ba0f13dfdfb)

Y ahora después de que se confirme la transacción de nuestra actualización de `NaiTokenV2` podremos ver el método `burn` activo.

![Graph](/im%C3%A1genes/v2.png)

