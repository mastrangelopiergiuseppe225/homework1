# Due homework di crittografia

## Homework 1 — App per la moltiplicazione scalare

Aprire `ecc-demo.html` in un browser: è un unico file HTML/CSS/JavaScript, funziona offline e non richiede installazione. Il caso iniziale è p=17, curva y²≡x³+7, G=(1,5), k=6. **Calcola** stampa tutti i multipli; **Anima** li mostra ogni 650 ms; **Ferma** interrompe l’animazione. Si possono scegliere tutti i punti finiti validi come base. Il grafico rappresenta punti discreti del campo finito, non una curva continua nel piano reale.

### Matematica e risultato

G è valido perché 5²≡8 e 1³+7≡8 modulo 17. La curva è non singolare: 4a³+27b²≡14≠0 modulo 17, con a=0 e b=7. Enumerando le coordinate si trovano 17 punti finiti; aggiungendo l’identità ∞ si ottiene un gruppo di ordine 18.

Per sommare P=(x₁,y₁) e R=(x₂,y₂), quando x₁≠x₂:

```
λ = (y₂−y₁) · (x₂−x₁)⁻¹ mod 17
x₃ = λ²−x₁−x₂ mod 17
y₃ = λ(x₁−x₃)−y₁ mod 17
```

Per raddoppiare un punto con y₁≠0, λ=3x₁²·(2y₁)⁻¹. L’inverso di d è l’elemento che soddisfa d·d⁻¹≡1. I casi speciali sono P+∞=P, P+(−P)=∞ e 2P=∞ se yP=0. Nel codice ∞ è `null`; il modulo normalizza anche i numeri negativi.

Esempio: per 2G, 10⁻¹≡12, λ≡3·12≡2; quindi x≡4−2=2 e y≡2(1−2)−5≡10.

| Multiplo | Punto |
|---|---|
| G | (1,5) |
| 2G | (2,10) |
| 3G | (5,9) |
| 4G | (12,1) |
| 5G | (12,16) |
| **6G** | **(5,8)** |
| 7G | (2,7) |
| 8G | (1,12) |
| 9G | ∞ |

La TOY PUBLIC KEY richiesta è dunque **Q=(5,8)**. G ha ordine 9, perciò (k+9)G=kG. Nell’app k=9 produce ∞: utile didatticamente, ma una chiave pubblica valida in un sistema reale non è l’identità.

### Implementazione

`add` gestisce identità, inversi, raddoppio e somma; `inverse` cerca un inverso nel piccolo campo; `multiply` implementa double-and-add. La visualizzazione usa somme successive, così stampa tutti i passi: O(k) operazioni di gruppo. Double-and-add calcola solo il risultato in O(log k), sfruttando la rappresentazione binaria di k. Mostrare k righe richiede comunque almeno O(k) lavoro. Gli interi JavaScript sono sufficienti per p=17; questa implementazione non va trasferita a secp256k1, che richiede aritmetica e implementazioni crittografiche appropriate.

### Esperimenti e interpretazione statistica

L’app enumera k=1,…,N e mostra frequenze dei punti, istogrammi di x e y, frequenze dei bit, correlazioni di Pearson e distanza di Hamming media. Il CSV contiene i singoli risultati. ∞ è contato come punto, ma escluso dalle coordinate e dai bit; le coppie di Hamming che lo coinvolgono sono escluse.

1. **G=(1,5), N=9:** ciascuno dei 9 elementi del sottogruppo compare una volta. Otto risultati sono finiti. Le x 1,2,5,12 compaiono due volte ciascuna; le y 1,5,7,8,9,10,12,16 una volta ciascuna. Le altre coordinate non compaiono.
2. **Stessa base, N=18:** gli stessi conteggi raddoppiano, incluso ∞. La distribuzione dei punti è uniforme sul sottogruppo, non sull’intera curva.
3. **Stessa base, N=8:** tutti i punti non identità del sottogruppo compaiono una volta. È la controparte toy di una chiave scelta uniformemente tra 1 e n−1.
4. **G=(3,0):** ordine 2; i multipli alternano (3,0) e ∞. Il comportamento dipende dal punto base, non solo dall’equazione.
5. **N non multiplo dell’ordine:** le frequenze differiscono perché l’ultimo ciclo è incompleto. Questo non segnala automaticamente un difetto crittografico.

La codifica usa 5 bit per x e 5 per y. Solo 17 dei 32 valori possibili sono coordinate ammesse, quindi pretendere un 50% di uno in ogni posizione sarebbe improprio; il bit più alto vale uno soltanto per 16. Pearson dipende dall’intervallo e dall’ordinamento numerico e non misura direttamente la sicurezza.

La distanza tra kG e (k+1)G illustra variazioni di codifica, ma incrementare k può cambiare più bit dell’input. Un test di avalanche più rigoroso confronterebbe input che differiscono in un solo bit, con campionamento ampio e trattamento delle chiavi valide. Anche quel test non dimostrerebbe la sicurezza della curva. La relazione (k+1)G=kG+G rimane sempre valida.

Qui invertire Q=kG richiede al massimo nove tentativi e restituisce k modulo 9. L’app dimostra l’operazione, **non** la difficoltà computazionale delle curve reali.

## Homework 2 — Dal mnemonic all’indirizzo Ethereum

Quella seguente è una pipeline comune per wallet HD con account Ethereum controllati da chiavi. Non tutti i wallet usano BIP-39, e gli account smart contract hanno altri meccanismi di controllo e indirizzamento.

```
Entropia casuale
  ↓ BIP-39: checksum e parole
Mnemonic + passphrase opzionale
  ↓ PBKDF2-HMAC-SHA512
Seed di 64 byte
  ↓ BIP-32: albero HD; BIP-44: scelta del percorso
Chiave privata k
  ↓ secp256k1: Q = kG
Chiave pubblica Q
  ↓ Keccak-256 di x || y; ultimi 20 byte
Indirizzo di 20 byte
  ↓ esadecimale e checksum EIP-55
Indirizzo 0x… con maiuscole/minuscole
```

### BIP-39 — Parole e seed

Si generano 128–256 bit di entropia, a passi di 32. Si aggiungono i primi ENT/32 bit di SHA-256(entropia). Gruppi di 11 bit selezionano parole da una lista di 2048 elementi: 128 bit danno 12 parole, 256 bit ne danno 24. Il mnemonic codifica casualità, non è una frase inventata dall’utente.

Il seed si deriva con PBKDF2-HMAC-SHA512: password=mnemonic, salt="mnemonic"+passphrase, 2048 iterazioni, risultato di 64 byte. Le stringhe usano UTF-8 e normalizzazione NFKD. La passphrase assente è la stringa vuota; ogni passphrase produce un seed, quindi una passphrase errata non viene riconosciuta automaticamente. Il checksum delle parole rileva alcuni errori, non autentica il proprietario. I 512 bit del seed non aumentano l’entropia iniziale: una derivazione deterministica non crea casualità nuova. [Specifica BIP-39](https://github.com/bitcoin/bips/blob/master/bip-0039.mediawiki).

### BIP-32 — Albero di chiavi

Dal seed si calcola HMAC-SHA512 con chiave ASCII "Bitcoin seed". La metà sinistra determina la chiave privata master, la destra il chain code di 32 byte. La stringa è prevista dallo standard anche quando l’albero serve per Ethereum.

Ogni chiave estesa comprende chiave e chain code. Per un figlio si calcola HMAC-SHA512, usando il chain code come chiave e dati che includono indice e chiave pubblica compressa del genitore; nella derivazione hardened si usa invece 0x00 seguito dalla chiave privata. La nuova chiave privata è (parse256(IL)+k_parent) mod n, e IR è il nuovo chain code. I casi invalidi previsti dallo standard vanno gestiti.

La derivazione normale permette di ottenere figli pubblici dalla chiave pubblica estesa; quella hardened richiede il genitore privato. Questo separa rami sensibili e limita alcuni rischi di esposizione delle chiavi. Il chain code è materiale di derivazione, non un saldo o un identificativo blockchain. [Specifica BIP-32](https://github.com/bitcoin/bips/blob/master/bip-0032.mediawiki).

### BIP-44 — Dove trovare una chiave

BIP-44 organizza l’albero con `m / purpose' / coin_type' / account' / change / address_index`. L’apostrofo indica derivazione hardened, con indice numerico i+2³¹. Un percorso Ethereum comune per il primo indirizzo è **m/44'/60'/0'/0/0**: purpose 44, coin type 60, account 0, ramo esterno 0, indice 0. Il valore 60 è registrato in SLIP-44. BIP-44 organizza la derivazione; non è una funzione hash. Wallet diversi possono usare convenzioni di percorso differenti: recuperare un account richiede mnemonic, eventuale passphrase e percorso corretto. [BIP-44](https://github.com/bitcoin/bips/blob/master/bip-0044.mediawiki), [registro SLIP-44](https://github.com/satoshilabs/slips/blob/master/slip-0044.md).

### secp256k1 — Chiave pubblica e firme

La curva è y²=x³+7 sul campo primo p=2²⁵⁶−2³²−977. Il generatore standard ha ordine primo
`n=0xFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFFEBAAEDCE6AF48A03BBFD25E8CD0364141` e cofattore 1. La chiave privata è un intero 1≤k<n; Q=kG è la chiave pubblica. p regola le coordinate, n gli scalari. [Parametri SEC 2, §2.4.1](https://www.secg.org/sec2-v2.pdf).

La difficoltà di recuperare k da Q è l’ECDLP; gli attacchi generici classici richiedono circa √n operazioni, nell’ordine di 2¹²⁸. L’ordine di grandezza non copre errori implementativi, perdite di seed o attacchi ai dispositivi. La moltiplicazione scalare è efficiente e iniettiva sulle chiavi valide, pur mostrando struttura algebrica.

Gli account Ethereum tradizionali usano firme ECDSA su secp256k1 per autorizzare transazioni: il segreto firma e la chiave pubblica consente la verifica. La firma è un’operazione distinta dal calcolo dell’indirizzo e richiede anche un nonce gestito correttamente. Il wallet gestisce le chiavi; fondi e stato sono sulla rete. [Account Ethereum](https://ethereum.org/en/developers/docs/accounts/).

### Keccak-256 — Dalla chiave all’indirizzo

Si serializza Q non compresso: prefisso 0x04, poi x e y su 32 byte ciascuna, big-endian. Per l’indirizzo si hashano **solo i 64 byte x∥y**, escludendo 0x04. Si prendono gli ultimi 20 byte, cioè i 160 bit meno significativi del digest Keccak-256. La rappresentazione esadecimale ha 40 caratteri più `0x`. Non si hasha il testo esadecimale della chiave, né la chiave pubblica compressa. [Account Ethereum](https://ethereum.org/en/developers/docs/accounts/).

SHA-512 appartiene a SHA-2 e produce 512 bit; Keccak ha una costruzione sponge, da cui deriva SHA-3. Ethereum usa **Keccak-256**, diverso da **SHA3-256** per la convenzione di padding/separazione di dominio: sostituirli produce indirizzi differenti. Anche la stessa lunghezza del digest non rende intercambiabili due hash. [Standard SHA-3, FIPS 202](https://www.nist.gov/publications/sha-3-standard-permutation-based-hash-and-extendable-output-functions).

### EIP-55 — Checksum tramite maiuscole

Si prende l’indirizzo esadecimale minuscolo, senza `0x`, e si calcola Keccak-256 dei suoi **caratteri ASCII**. Per ogni lettera a–f, si usa la maiuscola se il corrispondente nibble dell’hash è almeno 8; le cifre restano invariate. Si aggiunge infine `0x`.

Qui si hasha testo, mentre nel passaggio precedente si hashavano byte di coordinate. EIP-55 cambia la grafia, non i 20 byte dell’indirizzo. È un controllo di errori di trascrizione, non una firma, un’autenticazione o un’aggiunta di segretezza; un attaccante può ricalcolarlo. [Specifica EIP-55](https://eips.ethereum.org/EIPS/eip-55).

### Tecniche diverse, scopi diversi

| Categoria | Funzione nella pipeline | Proprietà da interpretare |
|---|---|---|
| Generazione | Campiona entropia o una chiave con casualità sicura | Imprevedibilità e assenza di bias rilevanti |
| Derivazione | Da materiale segreto esistente ottiene seed e figli | Deterministica; non crea entropia |
| Crittografia pubblica | Q=kG e firme | ECDLP, correttezza e sicurezza delle firme |
| Hashing | SHA-256, SHA-512/HMAC, Keccak | Preimmagini, seconde preimmagini, collisioni; diffusione dei bit |
| Checksum | Checksum mnemonic, EIP-55 | Rilevazione accidentale degli errori, senza autenticazione |
| Codifica | Parole, byte serializzati, esadecimale | Rappresentazione del dato; da sola non protegge segreti |

Per un hash ideale da 256 bit, il costo generico di preimmagine è circa 2²⁵⁶ e quello di collisione circa 2¹²⁸. Per un hash troncato a 160 bit, questi valori diventano circa 2¹⁶⁰ e 2⁸⁰: cercare una coppia qualsiasi di indirizzi collidenti è diverso dal colpire uno specifico indirizzo. Le stime sono modelli generici, non una dimostrazione della sicurezza dell’intera pipeline.

Nei buoni hash ci si aspetta, su campioni appropriati, bit approssimativamente bilanciati e circa metà dei bit di output cambiati quando si altera un bit dell’input. Questi test possono scoprire difetti evidenti, ma non provano resistenza crittografica. La compressione di input arbitrari implica collisioni, anche se trovarle deve essere difficile.

Per Q=kG è fondamentale l’ECDLP, non che ogni coordinata sembri un hash. Un k uniforme in 1,…,n−1 produce punti uniformi sui punti non identità del sottogruppo; la distribuzione delle codifiche coordinate risente della curva. BIP-39 conserva vincoli di checksum; i seed derivati possono apparire casuali senza avere 512 bit di entropia. Chiavi HD diverse devono essere difficili da prevedere senza il materiale richiesto, ma appartengono allo stesso albero deterministico. EIP-55 ha vincoli intenzionali e non va valutato come output casuale indipendente.

Un solo algoritmo non sostituisce tutto: un hash non fornisce la relazione pubblica/privata necessaria alle firme; una curva non offre il backup mnemonico o una gerarchia interoperabile; una codifica non rende il dato segreto; un checksum non autorizza una transazione.

### Contesto temporale

| Data | Tecnologia | Contesto |
|---|---|---|
| 1977 | RSA | Introduzione di un sistema di crittografia pubblica e firme; non è la curva usata negli account Ethereum |
| 1985–1987 | ECC | Proposte di Miller e Koblitz; il lavoro di Koblitz viene pubblicato nel 1987 |
| 2001–2002 | SHA-2, incluso SHA-512 | La consegna indica 2001; più precisamente la proposta di revisione è del 2001 e FIPS 180-2 viene pubblicato nel 2002 |
| 2012 | BIP-32 | Wallet gerarchici deterministici; Keccak viene selezionato per SHA-3 nello stesso anno |
| 2013 | BIP-39 | Backup delle informazioni segrete con parole |
| 2014 | BIP-44 | Convenzione per alberi con monete e account multipli |
| 2015 | SHA-3 / Ethereum | FIPS 202 standardizza SHA-3; Ethereum Frontier parte il 30 luglio |
| 2016 | EIP-55 | Checksum degli indirizzi tramite maiuscole/minuscole |

Le date BIP/EIP sono quelle delle rispettive proposte, non la data in cui ogni wallet le ha adottate. La cronologia non è l’ordine di esecuzione della pipeline. [Storia RSA](https://www.rsa.com/company/rsa-cryptography/), [bozza NIST del 2001](https://csrc.nist.gov/news/2001/announcing-draft-fips-180-2-and-request-for-commen), [approvazione FIPS 180-2 del 2002](https://csrc.nist.gov/news/2002/announcing-approval-of-fips-180-2-a-revision-of-f), [pubblicazioni di Koblitz](https://sites.math.washington.edu/~koblitz/pbl.html), [Miller, CRYPTO ’85](https://doi.org/10.1007/3-540-39799-x_31), [lancio Ethereum](https://blog.ethereum.org/2015/07/30/ethereum-launches). Le specifiche BIP ed EIP sopra citate riportano le date.

### Piccolo esperimento con un wallet

L’uso di un wallet è facoltativo nella consegna. Per studiarlo si può creare un wallet separato e senza fondi, annotare il percorso documentato e osservare che un altro indice produce un altro indirizzo; un ripristino con medesimo mnemonic, passphrase e percorso riproduce la stessa chiave. Confrontare lo stesso indirizzo minuscolo e in formato EIP-55 mostra che il dato binario è identico. Questa consegna non include un esperimento eseguito dentro MetaMask o altri wallet, né genera mnemonic reali: l’app implementa la curva toy e la relazione descrive la pipeline reale.
