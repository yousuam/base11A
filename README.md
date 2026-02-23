# base11Aimport time
from web3 import Web3

RPC_URL = "https://mainnet.base.org"

# ERC20 Transfer event topic
# keccak256("Transfer(address,address,uint256)")
TRANSFER_TOPIC = "0xddf252ad1be2c89b69c2b068fc378daa952ba7f163c4a11628f55a5f7b3b1a"

# ---- CONFIG ----
WATCH_WALLET = Web3.to_checksum_address("0xYOUR_WALLET_HERE")
MIN_AMOUNT_THRESHOLD = 10_000  # raw token units after decimals conversion
# ----------------

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


def safe_call(fn, fallback):
    try:
        return fn()
    except Exception:
        return fallback


def decode_address(topic_hex):
    return Web3.to_checksum_address("0x" + topic_hex[-40:])


def main():
    w3 = Web3(Web3.HTTPProvider(RPC_URL))
    if not w3.is_connected():
        raise RuntimeError("Cannot connect to Base RPC")

    print("Connected to Base")
    print("Watching wallet:", WATCH_WALLET)

    last_block = w3.eth.block_number

    while True:
        current = w3.eth.block_number

        if current > last_block:
            for b in range(last_block + 1, current + 1):

                logs = w3.eth.get_logs(
                    {
                        "fromBlock": b,
                        "toBlock": b,
                        "topics": [TRANSFER_TOPIC],
                    }
                )

                for log in logs:

                    from_addr = decode_address(log["topics"][1].hex())
                    to_addr = decode_address(log["topics"][2].hex())

                    if from_addr != WATCH_WALLET and to_addr != WATCH_WALLET:
                        continue

                    token_addr = Web3.to_checksum_address(log["address"])
                    token = w3.eth.contract(
                        address=token_addr, abi=ERC20_ABI_MIN
                    )

                    symbol = safe_call(
                        lambda: token.functions.symbol().call(),
                        "UNKNOWN",
                    )
                    decimals = safe_call(
                        lambda: token.functions.decimals().call(),
                        18,
                    )

                    raw_value = int(log["data"], 16)
                    value = raw_value / (10 ** decimals)

                    if value < MIN_AMOUNT_THRESHOLD:
                        continue

                    tx = log["transactionHash"].hex()

                    direction = "IN" if to_addr == WATCH_WALLET else "OUT"

                    print(f"🐋 LARGE TRANSFER | Block {b}")
                    print(f"Token: {symbol}")
                    print(f"Amount: {value}")
                    print(f"Direction: {direction}")
                    print(f"Tx: {tx}\n")

            last_block = current

        time.sleep(2)


if __name__ == "__main__":
    main()
