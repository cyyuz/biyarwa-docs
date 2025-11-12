# Test steps

## 1. 创建私钥

默认在 `~/.accounts.json` 文件中生成 40 个账户私钥，这些账户将作为交易的源账户，用于发起交易。

```go
package main

import (
	"encoding/hex"
	"encoding/json"
	"fmt"
	"os"
	"path/filepath"

	"github.com/InjectiveLabs/sdk-go/chain/crypto/ethsecp256k1"
)

var (
	accountCount = 40               // number of accounts to generate
	fileName     = ".accounts.json" // file name to save private keys (in user home directory)
)

func generatePrivateKey() (string, error) {
	privKey, err := ethsecp256k1.GenerateKey()
	if err != nil {
		return "", err
	}

	privKeyBytes := privKey.Bytes()
	privKeyHex := hex.EncodeToString(privKeyBytes)

	return privKeyHex, nil
}

func savePrivateKeysToFile(privateKeys []string, filePath string) error {
	data, err := json.MarshalIndent(privateKeys, "", "  ")
	if err != nil {
		return fmt.Errorf("failed to marshal config: %w", err)
	}

	dir := filepath.Dir(filePath)
	if err := os.MkdirAll(dir, 0755); err != nil {
		return fmt.Errorf("failed to create directory: %w", err)
	}

	if err := os.WriteFile(filePath, data, 0600); err != nil {
		return fmt.Errorf("failed to write file: %w", err)
	}

	return nil
}

func main() {
	homeDir, err := os.UserHomeDir()

	if err != nil {
		panic(fmt.Errorf("failed to get user home directory: %w", err))
	}
	configFilePath := filepath.Join(homeDir, fileName)

	var privateKeys []string

	for i := 0; i < accountCount; i++ {
		privKey, err := generatePrivateKey()
		if err != nil {
			fmt.Printf("failed to generate private key %d: %v\n", i+1, err)
			continue
		}
		privateKeys = append(privateKeys, privKey)
	}

	if err := savePrivateKeysToFile(privateKeys, configFilePath); err != nil {
		panic(fmt.Errorf("failed to save config file: %w", err))
	}
	fmt.Println("generate private keys successfully, save to", configFilePath)
}
```

## 2. 初始化账户

以有余额的创世账号作为源账户，向新生成的账号转移 10^18 单位原生代币，为后续交易提供代币支持。

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"path/filepath"
	"time"

	"cosmossdk.io/math"
	rpchttp "github.com/cometbft/cometbft/rpc/client/http"
	sdktypes "github.com/cosmos/cosmos-sdk/types"
	txtypes "github.com/cosmos/cosmos-sdk/types/tx"
	banktypes "github.com/cosmos/cosmos-sdk/x/bank/types"

	chainclient "github.com/InjectiveLabs/sdk-go/client/chain"
	"github.com/InjectiveLabs/sdk-go/client/common"
)

const (
	senderPriv     = "67C96704DD88F94DEFF657CD1EA917F152C8B70672F74AD39C89C8841F2A4691" // private key of sender
	transferAmount = 1000000000000000000                                                // 1 INJ (18 decimal places)
)

func loadPrivateKeysFromFile() ([]string, error) {
	homeDir, err := os.UserHomeDir()
	if err != nil {
		return nil, fmt.Errorf("failed to get user home directory: %w", err)
	}

	configFile := filepath.Join(homeDir, ".accounts.json")
	data, err := os.ReadFile(configFile)
	if err != nil {
		return nil, fmt.Errorf("failed to read config file: %w", err)
	}

	var privateKeys []string
	if err := json.Unmarshal(data, &privateKeys); err != nil {
		return nil, fmt.Errorf("failed to parse config file: %w", err)
	}

	return privateKeys, nil
}

func main() {
	privKeys, err := loadPrivateKeysFromFile()
	if err != nil {
		panic(fmt.Errorf("failed to load private keys: %w", err))
	}

	log.Printf("successfully loaded %d private keys, start transferring...", len(privKeys))

	network := common.LoadNetwork("local", "")
	tmClient, err := rpchttp.New(network.TmEndpoint)
	if err != nil {
		panic(fmt.Errorf("failed to create tm client: %w", err))
	}

	senderAddress, cosmosKeyring, err := chainclient.InitCosmosKeyring("", "", "", "", "",
		senderPriv, false,
	)
	if err != nil {
		panic(fmt.Errorf("failed to initialize sender account: %w", err))
	}

	clientCtx, err := chainclient.NewClientContext(
		network.ChainId,
		senderAddress.String(),
		cosmosKeyring,
	)
	if err != nil {
		panic(fmt.Errorf("failed to create client context: %w", err))
	}
	clientCtx = clientCtx.WithNodeURI(network.TmEndpoint).WithClient(tmClient)

	chainClient, err := chainclient.NewChainClientV2(
		clientCtx,
		network,
	)
	if err != nil {
		panic(fmt.Errorf("failed to create chain client: %w", err))
	}

	ctx, cancel := context.WithTimeout(context.Background(), 30*time.Second)
	defer cancel()

	gasPrice := chainClient.CurrentChainGasPrice(ctx)
	gasPrice = int64(float64(gasPrice) * 1.1) // increase 10% to avoid price fluctuation
	chainClient.SetGasPrice(gasPrice)

	for i := 0; i < len(privKeys); i++ {
		toAddress, _, err := chainclient.InitCosmosKeyring("", "", "", "", "",
			privKeys[i], false,
		)
		if err != nil {
			log.Printf("failed to load receiver account %d, skip transferring: %v", i, err)
			continue
		}

		msg := banktypes.MsgSend{
			FromAddress: senderAddress.String(),
			ToAddress:   toAddress.String(),
			Amount: []sdktypes.Coin{{
				Denom:  "inj",
				Amount: math.NewInt(transferAmount),
			}},
		}

		_, _, err = chainClient.BroadcastMsg(ctx, txtypes.BroadcastMode_BROADCAST_MODE_ASYNC, &msg)
		if err != nil {
			log.Printf("failed to transfer to account %d: %v", i, err)
			continue
		}
	}

	log.Printf("transfer completed, processed %d accounts", len(privKeys))
}

```

## 3. bank交易测试

程序接收 “节点编号” 参数（对应 4 个节点部署场景，取值范围为 0\~3），每个节点将从步骤 1 生成的 40 个账户中，选取与编号对应的 10 个账户作为交易原账户，用于发起交易。

```go
package main

import (
	"context"
	"encoding/json"
	"fmt"
	"log"
	"os"
	"path/filepath"
	"strconv"
	"sync"
	"time"

	"cosmossdk.io/math"
	rpchttp "github.com/cometbft/cometbft/rpc/client/http"
	sdktypes "github.com/cosmos/cosmos-sdk/types"
	txtypes "github.com/cosmos/cosmos-sdk/types/tx"
	banktypes "github.com/cosmos/cosmos-sdk/x/bank/types"

	chainclient "github.com/InjectiveLabs/sdk-go/client/chain"
	"github.com/InjectiveLabs/sdk-go/client/common"
)

const (
	timeout   = 1000 * time.Second // timeout for ctx
	loopCount = 20000              // loop count for each account
	gasLimit  = 200000             // gas limit for each transaction (max 30000000)
	toAddress = "inj1k2z3chspuk9wsufle69svmtmnlc07rvw9djya7"
)

var (
	totalSentTx  uint64 // total number of transactions sent (async broadcast successful)
	txCountMutex sync.Mutex
)

func loadPrivateKeysFromFile() ([]string, error) {
	homeDir, err := os.UserHomeDir()
	if err != nil {
		return nil, fmt.Errorf("failed to get user home directory: %w", err)
	}

	configFile := filepath.Join(homeDir, ".accounts.json")
	data, err := os.ReadFile(configFile)
	if err != nil {
		return nil, fmt.Errorf("failed to read config file: %w", err)
	}

	var privateKeys []string
	if err := json.Unmarshal(data, &privateKeys); err != nil {
		return nil, fmt.Errorf("failed to parse config file: %w", err)
	}

	return privateKeys, nil
}

func main() {
	if len(os.Args) < 2 {
		panic("Usage: go run main.go <batch_number>")
	}

	batchNum, err := strconv.Atoi(os.Args[1])
	if err != nil {
		panic(fmt.Errorf("batch number must be a number: %w", err))
	}

	if batchNum < 0 || batchNum >= 4 {
		panic("batch number must be between 0 and 3 (4 nodes total)")
	}

	allPrivKeys, err := loadPrivateKeysFromFile()
	if err != nil {
		panic(fmt.Errorf("failed to load private keys: %w", err))
	}

	startIndex := batchNum * 10
	endIndex := startIndex + 10

	if startIndex >= len(allPrivKeys) {
		panic(fmt.Errorf("batch %d out of range: private keys total only %d", batchNum, len(allPrivKeys)))
	}

	if endIndex > len(allPrivKeys) {
		endIndex = len(allPrivKeys)
	}

	privKeys := allPrivKeys[startIndex:endIndex]

	log.Printf("Load %d private keys (batch %d: index %d-%d), start transfer...", len(privKeys), batchNum, startIndex, endIndex-1)

	network := common.LoadNetwork("local", "")
	grpcPort := 9900 + batchNum
	network.ChainGrpcEndpoint = fmt.Sprintf("localhost:%d", grpcPort)
	log.Printf("gRPC port: %d", grpcPort)

	startTime := time.Now()

	var wg sync.WaitGroup

	for i := 0; i < len(privKeys); i++ {
		wg.Add(1)
		go func(accountIdx int) {
			defer wg.Done()
			privKey := privKeys[accountIdx]

			if err := runAccountTransferLoop(network, privKey); err != nil {
				log.Printf("%s transfer failed: %v", privKey, err)
			}
		}(i)
	}

	wg.Wait()
	totalDuration := time.Since(startTime)
	tps := float64(totalSentTx) / totalDuration.Seconds()
	log.Printf("total duration: %v", totalDuration.Round(time.Millisecond))
	log.Printf("total number of transactions sent: %d", totalSentTx)
	log.Printf("average TPS (number of transactions sent per second): %.2f", tps)

	log.Println("finish!")

}

func runAccountTransferLoop(network common.Network, privKey string) error {
	senderAddress, cosmosKeyring, err := chainclient.InitCosmosKeyring(
		"", "", "", "", "", privKey, false,
	)
	if err != nil {
		return fmt.Errorf("failed to initialize source account: %w", err)
	}

	tmClient, err := rpchttp.New(network.TmEndpoint)
	if err != nil {
		return fmt.Errorf("failed to create tm client: %w", err)
	}

	clientCtx, err := chainclient.NewClientContext(
		network.ChainId,
		senderAddress.String(),
		cosmosKeyring,
	)
	if err != nil {
		return fmt.Errorf("failed to create client context: %w", err)
	}

	clientCtx = clientCtx.WithNodeURI(network.TmEndpoint).WithClient(tmClient).WithSimulation(false)
	chainClient, err := chainclient.NewChainClientV2(
		clientCtx,
		network,
	)
	if err != nil {
		return fmt.Errorf("failed to create chain client: %w", err)
	}

	ctx, cancel := context.WithTimeout(context.Background(), timeout)
	defer cancel()
	gasPrice := chainClient.CurrentChainGasPrice(ctx)

	accNum, accSeq := chainClient.GetAccNonce()

	msg := banktypes.MsgSend{
		FromAddress: senderAddress.String(),
		ToAddress:   toAddress,
		Amount: []sdktypes.Coin{{
			Denom: "inj", Amount: math.NewInt(1)}, // 1 inj
		},
	}

	for i := 0; i < loopCount; i++ {
		txBytes, err := chainClient.BuildSignedTx(
			ctx, accNum, accSeq, gasLimit, uint64(gasPrice), &msg,
		)
		if err != nil {
			log.Printf("sign failed: %v", err)
			continue
		}

		_, err = chainClient.BroadcastSignedTx(ctx, txBytes, txtypes.BroadcastMode_BROADCAST_MODE_SYNC)
		if err != nil {
			log.Printf("broad cast failed: %v", err)
			continue
		}

		txCountMutex.Lock()
		totalSentTx++
		txCountMutex.Unlock()
		accSeq++
	}

	return nil
}
```

## 4. 合约测试

### 4.1 部署合约

参考 hackquest injective 部署合约教程。

### 4.2 发行代币

合约owner向步骤1创建的账号发行代币，用于后续的交易。

合约 Owner 账户向步骤 1 中创建的账户发行代币后续交易场景提供基础资金支持。

```go
```

### 4.3 转移代币&#x20;

1 ddd
