# CyberSec - Crypt

Pedro Henrique Ghiotto

Guilherme Wisniewski

Karys Barbosa

Nathalia Brasilino Gimenes

Matheus Mazieiro

Matheus Sousa

Matheus Cassatti

Alex Alves Cardoso

---

# **Summary of the Challenge**

- É dado um uma cifra SPN com 4 rounds.
- Nós também sabemos os *sbox* e o *pbox*.
- Precisamos descobrir a chave.
- Temos $2^{16}$ pares (plaintext, ciphertext).

---

### **SPN (substitution–permutation network)**

- Criptografia simétrica
- Input:
    - plain text
    - chave
- Rounds alternados de:
    - Substitution boxes (S-boxes)
        - Substituição não linear
        - Feita por blocos
        - Introduz confusão
    - Permutation boxes (P-boxes)
        - Espalha os bits adjacentes pelo comprimento do texto
        - Adiciona difusão
        
        ![image.png](image.png)
        

---

net.py

```python
import os
import socketserver
import string
import threading
from time import *
import time
import binascii

ROUNDS = 4
BLOCK_SIZE = 8

sbox = [237, 172, 175, 254, 173, 168, 187, 174, 53, 188, 165, 166, 161, 162, 131, 227, 191, 152, 63, 182, 169, 136, 171, 184, 149, 148, 183, 190, 181, 177, 163, 186, 207, 140, 143, 139, 147, 138, 155, 170, 134, 132, 135, 18, 193, 128, 129, 130, 157, 156, 151, 158, 153, 24, 154, 11, 141, 144, 21, 150, 146, 145, 179, 22, 245, 124, 236, 206, 105, 232, 43, 194, 229, 244, 247, 242, 233, 224, 235, 96, 253, 189, 219, 234, 241, 248, 251, 226, 117, 252, 213, 246, 240, 176, 249, 178, 205, 77, 231, 203, 137, 200, 107, 202, 133, 204, 228, 230, 225, 196, 195, 198, 201, 221, 199, 95, 216, 217, 159, 218, 209, 214, 215, 222, 83, 208, 211, 243, 44, 40, 46, 142, 32, 36, 185, 42, 45, 38, 47, 34, 33, 164, 167, 98, 41, 56, 55, 126, 57, 120, 59, 250, 37, 180, 119, 54, 52, 160, 51, 58, 5, 14, 79, 30, 8, 12, 13, 10, 68, 0, 39, 6, 1, 16, 3, 2, 23, 28, 29, 31, 27, 9, 7, 62, 4, 60, 19, 20, 48, 17, 87, 26, 239, 110, 111, 238, 109, 104, 35, 106, 101, 102, 103, 70, 49, 100, 99, 114, 61, 121, 223, 255, 88, 108, 123, 122, 84, 92, 125, 116, 112, 113, 115, 118, 197, 76, 15, 94, 73, 72, 75, 74, 81, 212, 69, 66, 65, 64, 97, 82, 93, 220, 71, 90, 25, 89, 91, 78, 85, 86, 127, 210, 80, 192, 67, 50]
perm = [1, 57, 6, 31, 30, 7, 26, 45, 21, 19, 63, 48, 41, 2, 0, 3, 4, 15, 43, 16, 62, 49, 55, 53, 50, 25, 47, 32, 14, 38, 60, 13, 10, 23, 35, 36, 22, 52, 51, 28, 18, 39, 58, 42, 8, 20, 33, 27, 37, 11, 12, 56, 34, 29, 46, 24, 59, 54, 44, 5, 40, 9, 61, 17]
key = open("flag.txt", "rb").read().strip()

class Service(socketserver.BaseRequestHandler):

    def key_expansion(self, key):
        keys = [None] * 5
        keys[0] = key[0:4] + key[8:12]
        keys[1] = key[4:8] + key[12:16]
        keys[2] = key[0:4] + key[8:12]
        keys[3] = key[4:8] + key[12:16]
        keys[4] = key[0:4] + key[8:12]
        return keys

    def apply_sbox(self, pt):
        ct = b''
        for byte in pt:
            ct += bytes([sbox[byte]])
        return ct

    def apply_perm(self, pt):
        pt = bin(int.from_bytes(pt, 'big'))[2:].zfill(64)
        ct = [None] * 64
        for i, c in enumerate(pt):
            ct[perm[i]] = c
        return bytes([int(''.join(ct[i : i + 8]), 2) for i in range(0, len(ct), 8)])

    def apply_key(self, pt, key):
        ct = b''
        for a, b in zip(pt, key):
            ct += bytes([a ^ b])
        return ct

    def handle(self):
        keys = self.key_expansion(key)
        for i in range(65536):
            pt = os.urandom(8)
            ct = pt
            ct = self.apply_key(ct, keys[0])
            for i in range(ROUNDS):
                ct = self.apply_sbox(ct)
                ct = self.apply_perm(ct)
                ct = self.apply_key(ct, keys[i+1])
            self.send(str((int.from_bytes(pt, 'big'), int.from_bytes(ct, 'big'))))

    def send(self, string, newline=True):
        if type(string) is str:
            string = string.encode("utf-8")

        if newline:
            string = string + b"\n"
        self.request.sendall(string)

    def receive(self, prompt="> "):
        self.send(prompt, newline=False)
        return self.request.recv(4096).strip()

class ThreadedService(
    socketserver.ThreadingMixIn,
    socketserver.TCPServer,
    socketserver.DatagramRequestHandler,
):
    pass

def main():

    port = 4004
    host = "0.0.0.0"

    service = Service
    server = ThreadedService((host, port), service)
    server.allow_reuse_address = True

    server_thread = threading.Thread(target=server.serve_forever)

    server_thread.daemon = True
    server_thread.start()

    print("Server started on " + str(server.server_address) + "!")

    # Now let the main thread just wait...
    while True:
        sleep(10)

if __name__ == "__main__":
    main()
```

### **Linear cryptanalysis**

- Olhe para o XOR entre 1 bit da entrada e 1 bit da saída. Eles são iguais?
    - Se eles forem iguais, então aquela chave vale 0
    - Caso contrário ela vale 1
- Em vez de olhar para um único bit, vamos olhar para um grupo de bits.
    - Chamaremos o grupo da entrada de `input mask`
    - Chamaremos o grupo da saída de `output mask`
    - Podemos (inteligentemente) considerar a paridade em vez de pensar no XOR de todos os bits
        - Se a paridade da entrada e a paridade da saída forem iguais, isso implica que a contribuição daquela chave no XOR global é equivalente a 0.
        - Caso contrário, a chave contribui com 1 no XOR global.
- Mas isso só funcionaria se `sbox` fosse linear
    - Não é o caso 😢
- Portanto precisamos de uma Tabela de Aproximação Linear (*Linear approximation table* - LAT)
    - Com certeza alguém já chamei isso de matriz de correlação
- O valor de uma célula é igual ao número de vezes em que a paridade foi verdadeira ao percorrer todas as entradas possíveis, menos [o maior valor de entrada dividido por 2]
    
    ```python
                             input_mask
                                                  
                   0     1     2     3     4     5     6 
    
             0     128,  0,    0,    0,    0,    0,    0,  
             1     0,   -90,   8,    14,  -2,   -4,   -2,
    output   2     0,    8,    94,  -10,  -2,    6,   -8,  ...
    mask     3     0,    14,   2,   -64,  -4,   -6,   -6, 
             4     0,   -6,    0,    6,   -86,  -12,  -2,
             5     0,    8,   -4,    8,    8,    56,  -8, 
                                 ...
    ```
    
- Como gerar LAT

```python
from tqdm import tqdm  # tqdm is optional but it is very nice in order to predict runtime

SBOX_INPUT_SIZE: int = 256
SBOX_OUTPUT_SIZE: int = 256

def create_lat(pn: PermutationNetwork) -> List[List[int]]:
    lat = [[-SBOX_INPUT_SIZE // 2 for _ in range(SBOX_INPUT_SIZE)] for _ in range(SBOX_OUTPUT_SIZE)]
    for input_mask in tqdm(range(SBOX_INPUT_SIZE)):
        for output_mask in range(SBOX_OUTPUT_SIZE):
            s = 0
            for value in range(SBOX_INPUT_SIZE):
                r = pn._single_sbox(value)
                s += (parity(output_mask & r) == parity(input_mask & value))
            lat[output_mask][input_mask] += s
    return lat
```

- Números muito grandes (ou pequenos) nessa matriz significa que o output é igual (ou diferente)
    - Isso é chamado de *bias*
    - Quanto maior o bias, mais a probabilidade que a *sbox* se comporte como linear
    - Se olharmos o bias de todos os `input masks` e `output masks` e só tem um único bit ativo, teremos um certo vetor como:
        
        ```python
        [-90, 94, -86, -92, 86, -98, 102, -96]
        ```
        
    - E as probabilidade correlacionadas são:
        
        ```python
        [0.1484375, 0.8671875, 0.1640625, 0.140625, 0.8359375, 0.1171875, 0.8984375, 0.125]
        ```
        
    - Nota: Quanto mais longe de $0.5$ melhor
- A paridade no fim deve ser igual a paridade no inicio
    - Se o bit estiver correto, conseguimos decifrar um round com sucesso
    - Caso contrário a paridade dará errado e a equação linear não vai ter o bias que deveria
        - Isso quer dizer que a aproximação linear não confere
- O código a seguir resolve 1 byte, que será armazenado em `guessed_key`
- Executar `pn.apply_perm(bytes([255] + [0] * 7))` nos dirá onde os bits desse byte estará na chave final

```python
# List of pairs plaintext, ciphertext on length greater than 64, in practice I made this 500, just to be on the safe side (we do have 2 ** 16 of those). 
paris = [(pt, ct)]  
# masks of the input and one stage before the last
input_bit, one_before_output_bit = b'\x10\x00\x00\x00\x00\x00\x00\x00', b'\x20\x00\x00\x00\x00\x00\x00\x00'

# Save all the biases
all_results = []
# Guessing 8 bits of key
for key_guess in tqdm(range(SBOX_OUTPUT_SIZE)):
    # Count the bias
    results = Counter()
    for pt, ct in pairs:
        # Decrypt the ciphertext
        unperm_ct = pn._rev_perm(ct)
        un_sbox = list(unperm_ct)
        # From now on we only care about the sbox in location 0
        un_sbox[0] = pn._rev_single_sbox(un_sbox[0] ^ key_guess)
        un_sbox = bytes(un_sbox)
        
        # byte_and perform and operation between two byte arrays, pairty calculates the pairty of the bits
        if parity(byte_and(pt, input_bit)) == parity(byte_and(un_sbox, one_before_output_bit)):
            results.update(['equal'])
        else:
            results.update(['not equal'])
    all_results.append(results)
# Find the largest biases
possible_keys = sorted([(r['equal'], i) for i, r in enumerate(all_results)] +
                       [(r['not equal'], i) for i, r in enumerate(all_results)], reverse=True)
print(possible_keys)
guessed_key = possible_keys[0][1]
```

- Fazer isso mais 7 vezes com os *payloads* corretos permitirá encontrar `keys[4]` e então voltar um estágio
    - Daí pra frente só para traz, literalmente:
        - Decifrar o nível anterior com a mesma ideia

---

### Referências

[https://github.com/yonlif/0x41414141-CTF-writeups/blob/main/SoulsPermutationNetwork.md](https://github.com/yonlif/0x41414141-CTF-writeups/blob/main/SoulsPermutationNetwork.md)

[https://rkm0959.tistory.com/195](https://rkm0959.tistory.com/195)
