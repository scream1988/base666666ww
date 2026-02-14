# base666666wwimport time
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

# Put your Uniswap V3 pool address here (Base mainnet)
POOL = Web3.to_checksum_address("0x0000000000000000000000000000000000000000")

# keccak256("Swap(address,address,int256,int256,uint160,uint128,int24)")
SWAP_TOPIC = "0xc42079f94a6350d7e6235f29174924f928cc2ac818eb64fed8004e115fbcca67"

POOL_ABI_MIN = [
    {
        "name": "token0",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "address"}],
    },
    {
        "name": "token1",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "address"}],
    },
]

ERC20_ABI_MIN = [
    {
        "name": "symbol",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "string"}],
    },
    {
        "name": "decimals",
        "type": "function",
        "stateMutability": "view",
        "inputs": [],
        "outputs": [{"name": "", "type": "uint8"}],
    },
]


def to_int256(b: bytes) -> int:
    v = int.from_bytes(b, "big", signed=False)
    if v >= 2**255:
        v -= 2**256
    return v


def safe_symbol(contract, fallback="???"):
    try:
        return contract.functions.symbol().call()
    except Exception:
        return fallback


def safe_decimals(contract, fallback=18):
    try:
        return contract.functions.decimals().call()
    except Exception:
        return fallback


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    if not w3.is_connected():
        raise RuntimeError("❌ Cannot connect to Base RPC")

    if POOL.lower() == "0x0000000000000000000000000000000000000000":
        raise RuntimeError("❌ Put a real Uniswap V3 pool address in POOL")

    pool = w3.eth.contract(address=POOL, abi=POOL_ABI_MIN)

    token0_addr = Web3.to_checksum_address(pool.functions.token0().call())
    token1_addr = Web3.to_checksum_address(pool.functions.token1().call())

    token0 = w3.eth.contract(address=token0_addr, abi=ERC20_ABI_MIN)
    token1 = w3.eth.contract(address=token1_addr, abi=ERC20_ABI_MIN)

    sym0 = safe_symbol(token0, "token0")
    sym1 = safe_symbol(token1, "token1")
    dec0 = safe_decimals(token0, 18)
    dec1 = safe_decimals(token1, 18)

    print("✅ Connected to Base")
    print("Pool:", POOL)
    print("token0:", token0_addr, f"({sym0}, decimals={dec0})")
    print("token1:", token1_addr, f"({sym1}, decimals={dec1})")

    last = w3.eth.block_number
    print("Starting block:", last)

    while True:
        try:
            current = w3.eth.block_number

            if current > last:
                for b in range(last + 1, current + 1):
                    logs = w3.eth.get_logs(
                        {
                            "fromBlock": b,
                            "toBlock": b,
                            "address": POOL,
                            "topics": [SWAP_TOPIC],
                        }
                    )

                    for log in logs:
                        raw = bytes.fromhex(log["data"][2:])

                        amount0_raw = to_int256(raw[0:32])
                        amount1_raw = to_int256(raw[32:64])

                        # Convert to human units
                        amount0 = amount0_raw / (10 ** dec0)
                        amount1 = amount1_raw / (10 ** dec1)

                        tx = log["transactionHash"].hex()

                        print(f"\nBlock {b} swap | tx={tx}")
                        print(f"{sym0}: {amount0}")
                        print(f"{sym1}: {amount1}")

                last = current

            time.sleep(2)

        except KeyboardInterrupt:
            print("\nStopped.")
            return

        except Exception as e:
            print("Error:", str(e))
            time.sleep(3)


if __name__ == "__main__":
    main()
