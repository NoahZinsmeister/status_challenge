# Status Coding Challenge: Software Engineer - Smart Contracts

A quick file guide: Payroll logic is found in [contracts/Payroll.sol](./contracts/Payroll.sol), the interface is in [contracts/PayrollInterface.sol](./contracts/PayrollInterface.sol). I implemented a [dummy EUR Token](./contracts/EURToken.sol) and an [oracle](./contracts/Oracle.sol) for testing purposes. Tests are in [test/payroll.js](./test/payroll.js).

### Solution Notes
While implementing this Payroll interface, my intent was to write clear and functional code. To clarify any remaining points of confusion, I've documented several key design features below.

* Employees are eligible to receive their prorated salary allotment starting 1 month from when they are added to the Payroll contract, and 1 month after all subsequent `payday` calls.
* Employees must choose a distribution over ERC20 tokens which they wish to receive their allotment in. The list of eligible ERC20 tokens is determined by the contract owner and can by viewed by calling `getWhitelist`. The contract _only_ pays out ERC20 tokens from this whitelist. Once added to the whitelist, tokens cannot be removed, nor can their addresses be changed. EUR is the default token (i.e. it's always in the whitelist).
* Employees may _not_ elect to receive their salary in ether (I made this restriction for the sake of simplicity). In the event that non-whitelisted tokens are sent to the contract (making them unrecoverable in normal circumstances), the owner may call `drainToken` to rescue the token balances. If anyone tries to transfer a non-whitelisted but ERC223-compliant token to the contract, `tokenFallback` ensures that the transfer will fail.
* At first glance, `calculatePayrollRunway` seems like it might return the number of days before the contract will be unable to fulfill a possible `payday` request. **This is not the case** - currently, `calculatePayrollRunway` sums all whitelisted token balances in units of EUR without accounting for employee-elected distributions over tokens. So, while the number of days given by `calculatePayrollRunway` is accurate if the contract's balances are appropriately distributed across tokens, they are not a time frame during which all `payday` requests are guaranteed to succeed. Again, this decision was made to cut down on code complexity.
* When sending tokens, I use `transfer` rather than `approve`. This is because implementing `calculatePayrollRunway` would have been harder if I had to keep track of potentially withdrawable employee allowances.
* The oracle's address is hard-coded address and immutable, like the tokens' (again, for simplicity). It is assumed that the Oracle updates all whitelisted token balances via `setExchangeRate` at acceptably fine-grained intervals, to ensure that EUR token exchange rates are kept current. In a real-world setting, one way to make this more practical/efficient would be for functions that require accurate exchange rates to fail if the time elapsed since `rateLastUpdated` for a given token was too high and emit an Event that would prompt the oracle to send the appropriate update(s).
* A note on the `EURExchangeRate`s sent by the oracle: Imagine two tokens, EUR and USD. Say the USD/EUR exchange rate is 3/1 (i.e. 3 USD trade for 1 EUR), but that EUR has 2 decimals while USD has 4. This means that 1 EUR is represented as 100 in the EUR contract, and 1 USD is represented as 10,000 in the USD contract. My code currently assumes that the oracle's `EURExchangeRate` value for USD/EUR is 3*10,000/100 = 300. This is a fairly fragile assumption, because we could imagine that the decimals were flipped, in which case the `EURExchangeRate` would be 3/100, which we would be forced to store as 0 or 1 since our rate variable is a `uint`. In general, the closer this exchange rate is to 1, the worse the Payroll performs, because of potential rounding issues in the `uint` operations performed in `calculatePayrollRunway` and `payday`. Similiar issues arise if yearly EUR salaries are small, e.g. on the order of 1e3. In a real-world setting, perhaps the easiest way around this fragility would be to add a decimals-like precision multiplier (say, 1e18) to some of the `uint` calculations or to the rate, and adjust the contract logic accordingly. For the sake of this exercise though, I have elected not to do so.


I'd be happy to comment on any of the above in more detail, or discuss additional features of my solution that I've omitted.