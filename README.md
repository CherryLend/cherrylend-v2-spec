<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
# Table of Contents

- [Smart Lending](#smart-lending)
  - [Introduction](#introduction)
  - [High-Level Overview](#high-level-overview)
  - [Smart Contract Technical Implementation](#smart-contract-technical-implementation)
    - [Get Loan](#get-loan)
    - [Repay Loan](#repay-loan)
    - [Cancel Loan Offer](#cancel-loan-offer)
    - [Liquidate Collateral](#liquidate-collateral)
    - [Collect Interest Payment](#collect-interest-payment)
  - [Splitting Loan Offer Into Multiple UTXOs And Selecting Loan UTXOs For Borrowers](#splitting-loan-offer-into-multiple-utxos-and-selecting-loan-utxos-for-borrowers)
- [Special Shout Outs](#special-shout-outs)
 
# Smart Lending
## Introduction
Smart Lending introduces an order book model for decentralized lending for the first time. This model offers a unique twist which highlights the best traits of pooled lending and peered lending, which we believe has the potential to create the best lending experience on Cardano. Here’s a simple explanation of its key features and why it stands out.

* **Order Book Model**: Unlike typical DeFi platforms that use liquidity pools, CherryLend V2 will use an order book model. This means lenders can place loan offers, and borrowers can take loans by either using partial offers or by putting together multiple offers. For instance, if a borrower wants to take a loan of $500, the system can fulfill this loan by partially using a $1000 offer leaving the rest available for others. Or a borrower could take a $2,000 loan for which the system puts two distinct $1,000 offers together. This offer selection is done algorithmically by the platform, so it works in a completely transparent and seamless manner for the end user. The specification for the selection algorithm will be made public once it’s completely defined.

* **Flexibility in Lending and Borrowing**: This fulfillment logic adds immense flexibility. Borrowers are not constrained to accept the fixed amounts of existing loan offers. They can borrow exactly what they need, making the process more efficient and tailored to individual requirements, while eliminating the need to use single UTxOs as in pooled lending, which natively generates contention.

* **Efficiency and Accessibility**: This lending model, utilizing multiple UTxOs, inherently mitigates transaction contention, thereby dispensing with the necessity for batchers, which are commonplace in pooled lending arrangements. Also, by being built on the Cardano blockchain, CherryLend V2 benefits from low transaction fees and high transaction speeds. This makes the lending process more accessible and cost-effective for a broader range of users.

* **Innovative Approach to DeFi**: The order book model applied to lending is an innovative approach in the DeFi space. It blends elements of traditional finance (like order books) with the benefits of the eUTxO model of Cardano. This offers more competitive rates with the ability for loans to be filled based on partial or multiple existing offers, and can lead to more competitive interest rates, as lenders compete to have their funds borrowed. Additionally, this should encourage increased liquidity since loans can be partially filled, making funds more readily available.


## High-Level Overview
The lender will have 4 actions they can perform.
* Create loan offers(Does not require any smart-contract validations)
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

Here's the proofread version with improvements for clarity and grammar:

The validators will be written in Aiken. There will be three main contracts.

A loan contract that holds the original loan offer UTXOs, handling the validation of lenders canceling a loan offer and a borrower receiving a loan. When the borrower receives the loan, they will send their collaterals to be locked up at the collateral contract.
A collateral contract that will handle the validations of lenders liquidating the collateral and borrowers getting back their collateral.
An interest script that will handle the validations of the lender getting their original loan amount and interest when the borrowers send the interest and original loan amount to this script to get their collaterals back.

## Smart Contract Technical Implementation
DISCLAIMER!!!! Sample Codes are simplified for easier reading. To view full code please visit https://github.com/CherryLend/cherrylend-v2-smart-contracts

### Get Loan
##### High-Level Implementation
To get a loan, we will consume multiple UTXOs from the validator script to fulfill a loan. We need to validate the output collateral UTXOs and any refund UTXOs. We will create a data type that has the following properties: loan amount, loan asset, collateral amount, collateral asset, interest amount, interest asset, lender address hash, and loan duration. We will do this by looping through both the outputs and inputs. We will then compare the resulting two data types.

We are looking at the datum value when the value specified in the datum is not being spent in the transaction, and look at the actual value in the UTXO when it is being spent. This ensures that the initial datum values are not corrupted (i.e., the interest amount and asset when the loan offer was submitted), and the requirements are not circumvented by changing the datum values in the output (i.e., the collateral datum value in the collateral UTXO).

To achieve this, we first need to fold the input UTXOs and remove duplicate lender address hashes. For duplicated lender address hashes, we will sum up the collateral amount, loan amount, and interest amount, and keep the loan duration the same. The collateral amount and interest amount will be calculated by looking at the datum values. The loan amount will be calculated by getting the value of the loan asset in the input UTXO.

We will assume that the output UTXO will be unique for each lender address, so we don't need to fold it. If it is not unique, the final validation will fail. We will use a filter map function. We will get the loan amount from the datum. The collateral amount will be calculated by grabbing the amount of collateral assets in the output UTXO. The map function will only return data from the output UTXO if the borrower address, lend time, total loan amount, and transaction ID in the datum are valid.

##### Quick Code Walkthrough
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
### Repay Loan
##### High-Level Implementation
To repay a loan, the borrower will consume the collateral locked up at the collateral script and send the interest payments to the interest script. Repaying loan validation will be similar to grabbing a loan. We will be creating a new data type(ValidateInterestPayment) that contains the following fields repay loan amount, repay loan asset, repay interest amount, repay interest asset, and lender address hash from looping through the inputs that are coming from the collateral script and outputs going back to the interest script. We will assume both the inputs from the collateral script and outputs going to the interest script both contain unique lender addresses so we don't need to fold here.  

First, we will use a map_filter through the inputs from the collateral script to grab the following information from the datum: lender address, loan amount, loan asset, repay interest amount, and repay interest asset. We know these values are valid because we already validated them in the loan script. We will only return the a constructed ValidateInterestPayment in the filter_map if the following validations are true: 
* The deadline to repay the loan has not passed, and the original loan borrower signed the transaction.
* The tx ids are the same. 
* The total loan amount and total repay loan amount, which we got from calculating the outputs going to the interest script,needs to equals the amount specified in the input collateral UTXOs datum.
* 
These checks ensure that the loans are from the same transaction, the borrower is paying off the entire loan, not just parts of it, and the borrower still has time to pay off the loan.

The output interest payment UTXOs will contain datum values specifying the loan asset and repay interest asset. We will map through each UTXOs then look into the UTXO to see how much assets it contains and return a constructed ValidateInterestPayment. 

The resulting two lists of ValidateInterestPayment from the inputs and outputs will confirm that the borrower is paying the correct amount back to the lenders. 
##### Quick Code Walkthrough
Grab all input UTXOs coming from the collateral validator 
```
let inputs_from_collateral_validator: List<Input> =
    get_inputs_from_script(ctx.transaction.inputs, collateral_stake_hash)
```
Grab all output UTXOs going to the interest validator
```
let outputs_to_interest_validator: List<Output> =
   find_script_outputs(ctx.transaction.outputs, interest_script_hash)
```

Get the total loan repay amount and interest amount from the UTXOs going to the interest validator
```
pub type LoanAndInterestAmount {
  loan_amount: Int,
  interest_amount: Int,
}

let total_repay_amount: LoanAndInterestAmount =
    get_total_repay_loan_and_interest_amount(outputs_to_interest_validator)

```
Get info from the inputs 

```
let inputs_collateral_info: List<ValidateRepayInfo> =
  get_inputs_collateral_info(
    inputs_from_collateral_validator,
    total_repay_amount,
    loan_tx_id,
    validity_range,
    signatures,
)

fn get_inputs_collateral_info(
  ...
) -> List<ValidateRepayInfo>
{
  list.filter_map(
    inputs_from_collateral_validator,
    fn(input_from_collateral_validator: Input) {
      ........
      let collateral_valid =
        tx_id_valid && total_loan_amount_valid && deadline_not_passed && signed_by_borrower &&                 total_interest_amount_valid
     if collateral_valid {
      None
    } else {
      Some(ValidateRepayInfo ....)
    }
    }
}
```
Get the output UTXOs info 
```
let outputs_interest_info: List<ValidateRepayInfo> =
  get_outputs_interest_info(outputs_to_interest_validator)
```
Check if the two info matches up 
```
let info_difference =
    list.difference(input_loan_info, outputs_collateral_info)
let info_matches = list.length(info_difference) == 0
info_matches
```


### Cancel Loan Offer
##### High-Level Implementation
To cancel a loan we need to make sure the original lender is signing the transaction. The original lender's address hash is stored in the datum of the loan offer UTXO.


##### Quick Code Walkthrough
Check transaction is signed by the lender
```
let must_be_signed_by_lender =
  list.has(ctx.transaction.extra_signatories, datum.lender_address_hash)
must_be_signed_by_lender
```

### Liquidate Collateral
##### High Level Implementation
To liquidate a collateral we need to first check that the repay period has expired and the original lender is signing the transaction. The collateral UTXOs datum contain the lend time and loan duration in its datum. The collateral UTXOs contains the lender's address hash in its datum. We simply need to check if the signatures in the transaction matches with the address hash in the datum.

##### Quick Code Walkthrough
Check the deadline has passed and transaction signed by original lender
```
let must_be_signed_by_lender =
  list.has(ctx.transaction.extra_signatories, datum.lender_address_hash)
let deadline_passed =
  datum.loan_duration + datum.lend_time < utils.get_lower_bound(
    ctx.transaction.validity_range,
  )
must_be_signed_by_lender && deadline_passed
```

### Collect Interest Payment
##### High Level Implementation
To cancel a loan we need to make sure the original lender is signing the transaction. The original lender's address hash is stored in the datum of the loan offer UTXO.


##### Quick Code Walkthrough
Check transaction has been signed by the original lender
```
let must_be_signed_by_lender =
  list.has(ctx.transaction.extra_signatories, datum.lender_address_hash)
must_be_signed_by_lender
```

### Splitting Loan Offer Into Multiple UTXOs And Selecting Loan UTXOs For Borrowers 
When a user submits a loan offer, we will split the initial loan into multiple small loans to be locked up at the loan script. Each split UTXO will contain an equal amount of assets, but the amount may vary for different tokens.

When a borrower searches for a loan, we will use an algorithm that gives priority to loan offers submitted earlier. The probability of a loan offer being selected increases exponentially with the time elapsed since the lender submitted it. Larger loan offers from lenders will have a higher chance of being selected, as they will contribute more UTXOs to the pool.

### Special Shout Outs
Special thanks to Keyan M's guidance, and open-sourced code from Anastasia Labs, Lenfi, and Sundae Labs for making this happen <3 


