---
DESCRIPTION: ENS on Solana (EoS) for Solana by Solana
---

# [EP5.3] [TempCheck] ENS On Solana 

  
  | **Status**                | TempCheck                    |
  | ------------------------- | ---------------------------- |
  | **Discussion Thread**     | []()                         |
  | **Discussion TempCheck**  |                              | 
  | **Votes**                 | []()                         |      


## Abstract
EIP-5559 and EIP-3668 in conjunction, known as CCIP-Write and CCIP-Read protocols respectively, are capable of securely writing to and reading from any blockchain or arbitrary storage. This in principle allows for a seamless native integration of ENS into any arbitrary on-chain or off-chain ecosystem. This draft outlines the proposal for a core ENS integration into Solana in particular, potential benefits to the ENS ecosystem and Solana ecosystems, and a substantial revenue stream for the ENS DAO.

## Motivation
Solana currently lacks a good native naming system due to the failure of Solana Naming Service (SNS). ENS in this case has an opportunity to develop horizontally across ecosystems, in particular into Solana ecosystem which currently ranks within top 5 in terms of market capitalisation and is beating Ethereum in DeFi TVL as well as numbers of total and daily active wallets. This proposal forsees ENS serving the users on the Solana blockchain through `s.eth` namespace on Ethereum mainnet which is currently forbidden by ENS registrar. A dedicated storage-oriented Solana program is foreseen to be deployed on Solana hosting the users' ENS records, which can be written to via CCIP-Write. The CCIP-Read-enabled Solana Resolver on Ethereum mainnet is tasked with the job of fetching the records from the program on Solana blockchain, performing any state verifications if necessary, and finally resolving it in Solana wallets like Phantom. Needless to say, ENS must be supported by native Solana wallets including EIP-3668 in the aforementioned outline. This foreseen construction of ENS on Solana (EoS) is of the form 'ENS on Solana (wallets) for Solana (users) by Solana (resolver)' since the ENS records are hosted on Solana to be used by Solana users in their Solana wallets.

## Specification
There are two ways of implementing the functionality described in the section above: 

1. Issue each `domain.s.eth` on Ethereum mainnet by wrapping `s.eth` with ENS Name Wrapper alongside necessary config that makes `domain.s.eth` an absolutely ownable ERC-1155 NFT with preset Solana resolver implementing CCIP-Read and CCIP-Write. These NFTs will be tradeable on Ethereum only.

2. Use ENSIP-10 wildcard resolver on `s.eth` with CCIP-Read support, which delegates the task of determining ownership of `domain.s.eth` to a Solana NFT program. The resulting NFTs will be tradeable locally on Solana only.

The latter method appears more suitable and cost efficient for native Solana target audience.

### Namespace
This proposal requires the DAO to release `s.eth` from the registrar for registration by ENS DAO through an executable vote.

### Pricing Modal
The following pricing mechanism is proposed:

| Domain          | Price 1      | Price 2      |
|----------------:|-------------:|-------------:|
| `12345.s.eth`   | `2$/year`    | `1$/year`    | 
| `1234.s.eth`    | `25$/year`   | `5$/year`    |
| `123.s.eth`     | `75$/year`   | `10$/year`   | 
| `12.s.eth`      | `100$/year`  | `25$/year`   | 
| `1.s.eth`       | `200$/year`  | `50$/year`   | 

This pricing model 1 adds roughly 66% to the DAO's revenue per year; given the current annualised revenue of $18 million at the moment, ENS DAO can expect additional $12 million/year from `*.s.eth` registrations and renewals. The pricing model 2 adds roughly 20% to the DAO's wallet amounting to annualised revenue of $4 million/year.

### Registration Tapering
In order to ensure equitable domain ownership and prevent squatting, this proposal suggests that registrations for `*.s.eth` be tapered in time according the following method:

1. Allow users to post `commit` for their desired registrations in 24 hour periods. Cap the commits at some value `total` for each period.

2. At the end of each 24 hour period, randomly select `N` commits from `total` to continue with their registration, and let the remaining `total - N` commits pass on into the next 24 hour period. 

3. Allow the users to fill the slots up to value `total` for the next round and repeat.

This proposal suggests the following values for epochs, `N` as function of labels' character lengths:

| `Epochs`      |       `N(1)` |      `N(12)` |      `N(123)` |     `N(1234)` |    `N(12345)` |      Revenue 1 |      Revenue 2 |
|--------------:|-------------:|-------------:|--------------:|--------------:|--------------:|---------------:|---------------:|
| `1st day`     |       `all`  |       `400`  |        `800`  |       `1200`  |       `8000`  | `$166,000/day` |  `$37,000/day` |
| `2-3 days`    |         `0`  |       `400`  |        `800`  |       `1200`  |       `8000`  | `$146,000/day` |  `$32,000/day` |
| `4-60 days`   |         `0`  |         `0`  |        `800`  |       `1200`  |       `8000`  | `$106,000/day` |  `$22,000/day` |   
| `61-90 days`  |         `0`  |         `0`  |         `0`   |       `1200`  |       `8000`  |  `$46,000/day` |  `$14,000/day` |
| `91-365 days` |         `0`  |         `0`  |         `0`   |          `0`  |       `8000`  |  `$16,000/day` |   `$8,000/day` |

where the `N(*)` values correspond to `N` as function of char length `*` and total is set such that `total = N × 2`. Using this tapering method, the DAO can expect an annualised revenue of nearly `$12,000,000` for price plan 1 or `$4,000,000` for price plan 2. 

> This tapering can be applied to both methods described above. Note that this is only an initial suggestion and the implementation is open to better ideas.

### Script
The following `python` script is useful if anyone wants to test their own custom price plans and suggest alternatives besides plan 1 and 2. Try it out in a simple IDE here: [`Script`](https://www.programiz.com/python-programming/online-compiler/)

```python
# Example price plan as an array; [1, 2, 3, 4, 5+] chars in $
price_plan = [200, 100, 75, 25, 2]

def calculate_revenue(price_plan):
    # Initialize the revenue chart
    revenue_chart = []
    # Define the tapering epochs, days in each epoch and corresponding N values
    epochs = ["1st day", "2-3 days", "4-60 days", "61-90 days", "91-365 days"]
    days = [1, 2, 57, 30, 275]
    N_values = [
        [100, 400, 800, 1200, 8000],
          [0, 400, 800, 1200, 8000],
            [0, 0, 800, 1200, 8000],
              [0, 0, 0, 1200, 8000],
                 [0, 0, 0, 0, 8000],
    ]
    # Calculate daily revenue and revenue for each epoch
    for i in range(len(epochs)):
        daily_revenue = sum(x * y for x, y in zip(N_values[i], price_plan))
        epoch_revenue = daily_revenue * days[i]
        revenue_chart.append((epochs[i], epoch_revenue, daily_revenue))
    return revenue_chart

def print_revenue_chart(revenue_chart):
    # Print the revenue chart header
    print("Epoch         |  Epoch Rev   |  Daily Rev")
    print("-----------------------------------------")
    yearly_revenue = sum(revenue for _, revenue, _ in revenue_chart)
    # Print revenue for each epoch
    for epoch, revenue, daily in revenue_chart:
        print(f"{epoch.ljust(13)} | ${revenue:,.0f}   | ${daily:,.0f}")
    print("Yearly Revenue:", f"${yearly_revenue:,.0f}")

# Calculate revenue chart
revenue_chart = calculate_revenue(price_plan)
# Print the revenue chart
print("Revenue Chart:")
print_revenue_chart(revenue_chart)
```

## Funding Request
SysStruct/NameSys is requesting a funding of `$500,000` for the duration of one year to work on this mission and make ENS truly multi-chain. The funding is divided into 5 tranches of `$100,000` each released upon completion of pre-specified milestones at the end of each quarter. The funding encompasses the following tasks:

1. Proposal introducing `StorageHandledBySolana()` revert to EIP-5559.
2. Collaborating with Solana Wallet Provider(s) to integrate ENS.
3. Collaborating with Solana Foundation, Solana's biggest NFT marketplace [`tensor.hq`](tensor.trade) and ENS Labs for a dedicated launch.
4. The entire end-to-end codebase including contracts, clients and any infrastructrual packages.

| Tranche         | Funds        | When?        |                Milestones |
|----------------:|-------------:|-------------:|--------------------------:|
| `0`             | `$100,000`   |   `-Q1`      |   `EIP-X`, `Full Specs`   | 
| `1`             | `$100,000`   | `Q1-Q2`      | `Prototype`, `Testnet`    |  
| `2`             | `$100,000`   | `Q2-Q3`      |     `Testing`, `Audit`    |  
| `3`             | `$100,000`   | `Q3-Q4`      |    `Mainnet`, `Launch`    |  
| `4`             | `$100,000`   | `Q4-`        |                  `End`    |  

## Next Steps
Following this TempCheck, a social proposal will be floated finalising the price plan and tapering methodology/parameters. Following success of social proposal, an executable proposal will be floated with the necessary calls to release `s.eth` from ENS root to ENS DAO.