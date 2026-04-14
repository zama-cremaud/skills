# React + wagmi setup

**Canonical doc:** `github.com/zama-ai/sdk/docs/gitbook/src/tutorials/quick-start.md` (React + wagmi tab)

```bash
pnpm add @zama-fhe/react-sdk @tanstack/react-query wagmi viem
```

All Zama symbols are re-exported from `@zama-fhe/react-sdk` — no need to import from `@zama-fhe/sdk` directly.

```tsx
import { WagmiProvider, createConfig, http } from "wagmi";
import { sepolia } from "wagmi/chains";
import { injected } from "wagmi/connectors";
import { QueryClient, QueryClientProvider } from "@tanstack/react-query";
import { ZamaProvider, RelayerWeb, SepoliaConfig, indexedDBStorage } from "@zama-fhe/react-sdk";
import { WagmiSigner } from "@zama-fhe/react-sdk/wagmi";

const wagmiConfig = createConfig({
  chains: [sepolia],
  connectors: [injected()],
  transports: { [sepolia.id]: http("https://sepolia.infura.io/v3/YOUR_KEY") },
});

const signer = new WagmiSigner({ config: wagmiConfig });
const relayer = new RelayerWeb({
  getChainId: () => signer.getChainId(),
  transports: {
    [SepoliaConfig.chainId]: {
      ...SepoliaConfig,
      relayerUrl: "https://your-app.com/api/relayer/11155111", // proxy via your backend
      network: "https://sepolia.infura.io/v3/YOUR_KEY",
    },
  },
});
const queryClient = new QueryClient();

<WagmiProvider config={wagmiConfig}>
  <QueryClientProvider client={queryClient}>
    <ZamaProvider relayer={relayer} signer={signer} storage={indexedDBStorage}>
      <App />
    </ZamaProvider>
  </QueryClientProvider>
</WagmiProvider>
```

`signer`, `relayer`, `wagmiConfig`, `queryClient` can all sit at module scope — no `useConfig()` indirection needed.

## High-level token ops

```tsx
import { useConfidentialBalance, useConfidentialTransfer, useShield } from "@zama-fhe/react-sdk";

const { data: balance } = useConfidentialBalance({ tokenAddress: TOKEN });
const { mutateAsync: shield } = useShield({ tokenAddress: TOKEN });
const { mutateAsync: transfer } = useConfidentialTransfer({ tokenAddress: TOKEN });

await shield({ amount: 1000n });
await transfer({ to: "0x...", amount: 500n });
```

## Custom FHE contracts (auctions, voting, non-token)

Use `useEncrypt` + `useUserDecrypt` directly. See `guides/encrypt-decrypt.md` for the full pattern.
