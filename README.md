
# Comunicação com Servidores de Habbo Usando TCP e Manipulação de Bytes

Este guia prático destina-se a desenvolvedores interessados em compreender e implementar a comunicação com servidores de Habbo, utilizando o protocolo TCP e manipulação de bytes. Antes de prosseguirmos, é crucial que você esteja equipado com os seguintes conhecimentos prévios:

- **Protocolo TCP:** Entendimento básico sobre o Protocolo de Controle de Transmissão e como implementá-lo na linguagem de programação de sua escolha. [Saiba mais sobre TCP](https://pt.wikipedia.org/wiki/Protocolo_de_Controle_de_Transmiss%C3%A3o).
- **Manipulação de Binários:** Familiaridade com conceitos como ordem dos bytes (Little Endian e Big Endian) e como manipulá-los em sua linguagem de programação, utilizando ferramentas como o `ByteBuffer` do Java. [Entenda a ordem dos bytes](https://pt.wikipedia.org/wiki/Extremidade_(ordena%C3%A7%C3%A3o)).

## Tipos de Dados na Comunicação

Ao comunicarmos com servidores de Habbo, usamos diversos tipos de dados para montar e desmontar pacotes. A tabela abaixo resume esses tipos e suas características:

| Tipo     | Largura   | Descrição |
|----------|-----------|-----------|
| Int      | 4 bytes   | Um inteiro padrão. |
| Short    | 2 bytes   | Um inteiro curto. |
| Boolean  | 1 byte    | Um valor verdadeiro ou falso. |
| String   | n bytes   | Uma sequência de caracteres, precedida por um `Short` que indica o tamanho da string em bytes, codificada em [UTF-8](https://pt.wikipedia.org/wiki/UTF-8). |

## Estrutura dos Pacotes

Todos os pacotes no Habbo seguem um formato padrão:

```
Largura  | Header | Dados
0000000a | 0a2e   | 0000000000000000
```

- **Largura:** Os primeiros 4 bytes indicam o tamanho do pacote, excluindo-se estes próprios 4 bytes. Por exemplo, `0000000a` indica que há 10 bytes a seguir.
- **Header:** Os próximos 2 bytes representam o identificador do pacote, permitindo a identificação do formato dos dados subsequentes.
- **Dados:** Seguem o header, variando conforme o tipo de pacote.

Os pacotes são escritos seguindo a ordem Big Endian, onde o byte mais significativo vem primeiro.

### Descobrindo a Estrutura dos Dados dos Pacotes

A estrutura dos dados pode ser encontrada no repositório [nitro-renderer](https://github.com/billsonnn/nitro-renderer), que contém arquivos especificando os nomes e headers dos pacotes [OUTGOING](https://github.com/billsonnn/nitro-renderer/blob/81cfd5c56fcc42e2edb1e5c6fdc1248690da9d5f/src/nitro/communication/messages/outgoing/OutgoingHeader.ts#L150) e [INCOMING](https://github.com/billsonnn/nitro-renderer/blob/81cfd5c56fcc42e2edb1e5c6fdc1248690da9d5f/src/nitro/communication/messages/incoming/IncomingHeader.ts). Ao identificar o header do pacote, busque pela classe correspondente no arquivo [NitroMessages](https://github.com/billsonnn/nitro-renderer/blob/81cfd5c56fcc42e2edb1e5c6fdc1248690da9d5f/src/nitro/communication/NitroMessages.ts) para entender como os dados são manipulados.

## Exemplo Prático: Handshake com Servidor Habbo

Para realizar um handshake com um servidor Habbo, siga os passos abaixo usando Typescript e NodeJS:

1. **Conectar ao Servidor:**

```typescript
import net from "net";

const connection = net.createConnection(30000, "habboretro.com", () => {
    console.log("Conectado.");
});
```

2. **Ler Dados do Servidor:**

```typescript
connection.on("data", data => {
    // Vamos ler os 4 primeiros bytes (32 bits) na ordem Big Endian, 0 é o offset
    const length = data.readInt32BE(0);
    // Aqui lemos os 2 bytes do header do pacote pulando os 4 primeiros bytes
    const header = data.readInt16BE(4);
});
```
3. Por ultimo você precisará enviar 3 pacotes necessários para a autenticação, o ReleaseVersion, MachineID e SecureTicket.

```typescript
// Essa função é para a criptografia do sso caso você esteja fazendo o bot para o Hotel Habblet, você irá precisar disso.
function crypt(e: string) {
  const t = (e: string) =>
    "ABCDEFGHIJKLMNOPQRSTUVWXYZabcdefghijklmnopqrstuvwxyz".indexOf(e);
  return e
    .split("")
    .map((e) =>
      t(e) > -1
        ? "NOPQRSTUVWXYZABCDEFGHIJKLMnopqrstuvwxyzabcdefghijklm"[t(e)]
        : e
    )
    .join("");
}

connection.on("connect", data => {
    // Vamos anotar os dados que irão ser escritos no buffer
    const production = Buffer.from("PRODUCTION-202101271337-HTML5");
    const type = Buffer.from("HTML5");
    const platform = 2;
    const category = 1;
    // Vamos criar um buffer contendo a quantia total de bytes
    let packet = Buffer.alloc(18);
    // Agora vamos montar o pacote
    packet.writeInt32BE(production.length + type.length + 14);
    packet.writeInt16BE(4000);
    packet.writeInt16BE(production.length);
    packet = Buffer.concat([packet, production]);
    packet.writeInt16BE(type.length);
    packet = Buffer.concat([packet, type]);
    packet.writeInt32BE(platform);
    packet.writeInt32BE(category);
    // Agora enviamos o packet de ReleaseVersion
    connection.write(packet);

    // Após o envio do primeiro pacote, vamos enviar agora o pacote SecureTicket
    // Esse "sso" é o ticket que você irá na página do hotel na aba do nitro ou flash
    const sso = Buffer.from("cole seu sso aqui");
    const ssoEncrypted = Buffer.from(crypt("cole seu sso aqui também"));
    // Esse é o tempo que demorou pro jogo iniciar, pode colocar o valor que quiser em milisegundos
    const time = 10_000;
    packet = Buffer.alloc(14);
    packet.writeInt32BE(sso.length + ssoEncrypted.length + 10);
    packet.writeInt16BE(2419);
    packet.writeInt16BE(sso.length);
    packet = Buffer.concat([packet, sso]);
    packet.writeInt32BE(time);
    packet.writeInt16BE(ssoEncrypted.length);
    packet = Buffer.concat([packet, ssoEncrypted]);
});
```
