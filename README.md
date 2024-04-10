<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
# Table of Contents

- [Smart Lending](#smart-lending)
  - [Introduction](#introduction)
  - [Smart Contract Design Framework](#smart-contract-desgin-framework)
  - [Technical Implementation](#technical-implementation)
    - [Grab Loan](#grab-loan)
        - [High Level](#grab-loan-high-level)
        - [Smart Contract Implementation](#grab-loan-validation)


  - [License](#license)
  - [Special Shout Outs](#special-shout-outs)
 
# Smart Lending

## Introduction
Smart Lending introduces an order book model for decentralized lending for the first time. This model offers a unique twist which highlights the best traits of pooled lending and peered lending, which we believe has the potential to create the best lending experience on Cardano. Here’s a simple explanation of its key features and why it stands out.

* **Order Book Model**: Unlike typical DeFi platforms that use liquidity pools, CherryLend V2 will use an order book model. This means lenders can place loan offers, and borrowers can take loans by either using partial offers or by putting together multiple offers. For instance, if a borrower wants to take a loan of $500, the system can fulfill this loan by partially using a $1000 offer leaving the rest available for others. Or a borrower could take a $2,000 loan for which the system puts two distinct $1,000 offers together. This offer selection is done algorithmically by the platform, so it works in a completely transparent and seamless manner for the end user. The specification for the selection algorithm will be made public once it’s completely defined.

* **Flexibility in Lending and Borrowing**: This fulfillment logic adds immense flexibility. Borrowers are not constrained to accept the fixed amounts of existing loan offers. They can borrow exactly what they need, making the process more efficient and tailored to individual requirements, while eliminating the need to use single UTxOs as in pooled lending, which natively generates contention.

* **Efficiency and Accessibility**: This lending model, utilizing multiple UTxOs, inherently mitigates transaction contention, thereby dispensing with the necessity for batchers, which are commonplace in pooled lending arrangements. Also, by being built on the Cardano blockchain, CherryLend V2 benefits from low transaction fees and high transaction speeds. This makes the lending process more accessible and cost-effective for a broader range of users.

* **Innovative Approach to DeFi**: The order book model applied to lending is an innovative approach in the DeFi space. It blends elements of traditional finance (like order books) with the benefits of the eUTxO model of Cardano. This offers more competitive rates with the ability for loans to be filled based on partial or multiple existing offers, and can lead to more competitive interest rates, as lenders compete to have their funds borrowed. Additionally, this should encourage increased liquidity since loans can be partially filled, making funds more readily available.


## Smart Contract Design Approach 
For this smart contract there will be two parties involved. A lender and a borrower. The lender will have 4 actions they can perform(1 not requiring any smart contract interaction).
* Create loan offers
* Cancel loan offers
* Liquidate collaterals
* Collect interests
  
The borrower will have 2 actions they can perform
* Get loan
* Repay loan

We want to make sure of the following things for the lenders
* For the amount of loan they gave out, there needs to be the correct amount of collateral locked up in the contract
* They will collect the correct amount of interest for the loan they gave out

We want to make sure of the following things for the borrowers
* They can get their collateral back once they pay back the correct amount of interest

There will be 3 different validators, and 2 different staking validators being used. We will be consuming multiple UTXOs from the script and for a more efficient transaction we will be using a staking validator for validating logics. Read more about that here [https://github.com/Anastasia-Labs/design-patterns/blob/main/stake-validator/STAKE-VALIDATOR.md]


The action of canceling a loan offer and grabbing a loan will be on one validator and an attached staking validator. When the user grabs a loan it will send the collateral to the collateral validator address.
The actions of liquidating collateral and or paying interest will be on one validator and an attached staking validator. When the user pays back interest it will be sent to the interest validator address.  
The actions of getting interest payment will be on one script validator


The actions of cancelling loans, liquidating collaterals, and grabbing interest payments will simply require looking at the signatures in the transaction. Then checking if it came from the address specified in the datum of the UTXOs they are consuming. 
A brief overview of the action getting a loan and repaying the loan requires us to validate the datum and value of the UTXOs in the outputs are valid in regards to the input UTXOs coming from the validator.  

## Technical Implementation

### Grab Loan
###### High Level Implementation
We are consuming multiple UTXOs from the validator scipt to fulfill a loan and we need to validate the output collateral UTXOs and any refund UTXOs are valid. We will create a data type that has properties such as collateral amount, interest amount etc.... from both the outputs and inputs. We will then compare the resulting two datatypes. 

To achieve this we first need to fold the input UTXOs to remove duplicate lender addresses and combine the collateral amount,loan amount, and interest amount. 

The collateral amount from inputs will be calculated by looking at the datum values and adding them. The interest amount will also be added up by looking at the datum. The loan amount will be calculated by grabbing the asset from the datum looking at each input's value and getting the amount of that asset in that UTXO. 

We will assume the output UTXO will be for each unique for each lender address so we don’t need to fold it. If it is not unique the final validation will still fail. We will map the output collateral UTXO and grab the value loan_amount from the datum, and the collateral amount by looking at the collateral asset and grabbing the amount of that asset in that UTXO. We will also add one additional logic in the map function. The map function will only return data from the output UTXO if the borrower address, lend time, total loan amount, and transaction id in the datum are valid. 

We are looking at the datum value when the value specified in the datum is not being spent in the transaction and look at the actual in the UTXO value when it is being spent. This ensures the initial datum values are not corrupted(ie the interest amount and asset when the loan offer was submitted), and the requirements of performing an action are not be gamed by changing the datum values in the output(ie the collateral datum value in the collateral UTXO).

###### High Level Code
Data type to compare the input and output UTXOs
```
pub type Info {
  loan_amount: Int,
  loan_asset: AssetClass,
  collateral_amount: Int,
  collateral_asset: AssetClass,
  interest_amount: Int,
  interest_asset: AssetClass,
  lender_address_hash: AddressHash,
  loan_duration: POSIXTime,
}
```
Grab all inputs from the validator that are being spent
```
let inputs_from_script_validator: List<Input> =
    find_script_outputs()
```

Create a function that folds all the loan UTXOs from the script and return a list of info

```
let inputs_loan_info: List<Info> = get_inputs_loan_info()
```

Grab all outputs that are going to the collateral address 

```
let outputs_to_collateral_validator: List<Output> = find_script_outputs()    
```

Find if there are any refund UTXOs going back to the loan validator script.

Create a function that loops through the outputs to collateral validator and any potential refund UTXOs to return a list of info. 

```
let outputs_collateral_info: List<Info> = outputs_collateral_info()
```

Check if the two info matches up 
```
  let info_difference =
    list.difference(input_loan_info, outputs_collateral_info)
  let info_matches = list.length(info_difference) == 0
  info_matches
```

### Special Shout Outs
Special thanks to Keyan M's guidance, and open-sourced code from Anastasia Labs and Lenfi for making this happen <3 


