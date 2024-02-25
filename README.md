Antes de tudo vamos nos certificar que você tenha os pre requisitos para o artigo.

- Este artigo espera que você tenha um conhecimento sobre o protocolo [TCP](https://pt.wikipedia.org/wiki/Protocolo_de_Controle_de_Transmiss%C3%A3o), e saiba como implementa-lo na linguagem de sua preferência.
- Conhecimento sobre binários, [ordem dos bytes](https://pt.wikipedia.org/wiki/Extremidade_(ordena%C3%A7%C3%A3o)) Little Endian e Big Endian, e também a manipulação deles, você pode usar algo como o ByteBuffer do Java.

---
# Tipos de dados
Abaixo está a tabela dos tipos de dados usados na montagem e desmontagem dos pacotes.
|             | Int         | Short       | Boolean     | String      |
|-------------|-------------|-------------|-------------|-------------|
| Lagura      | 4 bytes     | 2 bytes     | 1 byte      | n bytes     |
| Tipo        | int         | short int   | bool        |             |

O tipo `String` é um tipo especial pois ele é composto por um `Short` indicando a largura do texto em bytes, e em seguida temos a cadeia de bytes que irão representar a string codificados em [UTF-8](https://pt.wikipedia.org/wiki/UTF-8).

---

# Estrutura dos pacotes
Os pacotes do Habbo seguem um formato padrão em todos os pacotes.
```
Largura  Header  Dados
0000000a  0a2e   0000000000000000
```
Os primeiros 4 bytes é a quantidade de bytes que tem no pacote ignorando os primeiros 4 bytes, por exemplo se os primeiros 4 bytes forem `0000000a` isso significa que há 10 bytes a seguir para serem lidos. Os proximos 2 bytes é o header, o header é um identificador (id) do pacote, ou seja, cada pacote tem uma forma específica de ler os dados, que são os bytes após os 6 primeiros bytes. Os pacotes são escritos na ordem do byte mais significativo (Big Endian).

---

# Handshake

Finalmente chegamos em uma das partes mais legais, que é a parte em que nos comunicamos de verdade com um servidor de Habbo. Estarei usando a linguagem Typescript com NodeJS por ser de fácil leitura e compreensão do código.

1. Você deverá se conectar ao servidor de Habbo desejado como o exemplo no código typescript + nodejs abaixo.
```ts
import net from "net";

const connection = net.createConnection(30000, "habboretro.com", () => {
    console.log("Conectado.");
});
```
2. Espere por dados do servidor e leia a largura e o header do pacote.
```ts
connection.on("data", data => {
    // Vamos ler os 4 primeiros bytes (32 bits) na ordem Big Endian, 0 é o offset
     const length = data.readInt32BE(0);
     // Aqui lemos os 2 bytes do header do pacote pulando os 4 primeiros bytes
     const header = data.readInt16BE(4);
});
```
