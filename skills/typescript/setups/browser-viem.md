# Browser + viem setup

**Canonical doc:** `github.com/zama-ai/sdk/docs/gitbook/src/tutorials/quick-start.md` (viem tab)

```bash
pnpm add @zama-fhe/sdk viem
```

```ts
import { createPublicClient, createWalletClient, custom, http } from "viem";
import { sepolia } from "viem/chains";
import { ZamaSDK, RelayerWeb, SepoliaConfig, indexedDBStorage } from "@zama-fhe/sdk";
import { ViemSigner } from "@zama-fhe/sdk/viem";

const publicClient = createPublicClient({ chain: sepolia, transport: http("https://sepolia.infura.io/v3/YOUR_KEY") });
const walletClient = createWalletClient({ chain: sepolia, transport: custom(window.ethereum!) });
const signer = new ViemSigner({ walletClient, publicClient });

const sdk = new ZamaSDK({
  relayer: new RelayerWeb({
    getChainId: () => signer.getChainId(),
    transports: {
      [SepoliaConfig.chainId]: {
        ...SepoliaConfig,
        relayerUrl: "https://your-app.com/api/relayer/11155111", // proxy via your backend
        network: "https://sepolia.infura.io/v3/YOUR_KEY",
      },
    },
  }),
  signer,
  storage: indexedDBStorage,
});
```

## High-level token ops

```ts
const token = sdk.createToken("0xYourEncryptedERC20");
await token.shield(1000n);
const balance = await token.balanceOf();
await token.confidentialTransfer("0xRecipient", 500n);
await token.unshield(500n);
```

Vite needs COOP/COEP headers — see `skill.md` → Universal gotchas.
