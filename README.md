<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
# Table of Contents

- [Smart Lending](#smart-lending)
  - [Introduction](#introduction)
  - [High-Level Overview](#high-level-overview)
  - [Splitting Loan Offer Into Multiple UTXOs And Selecting Loan UTXOs For Borrowers](#splitting-loan-offer-into-multiple-utxos-and-selecting-loan-utxos-for-borrowers)
  - [Smart Contract Technical Implementation](#smart-contract-technical-implementation)
    - [Get Loan](#get-loan)
    - [Repay Loan](#repay-loan)
    - [Cancel Loan Offer](#cancel-loan-offer)
    - [Liquidate Collateral](#liquidate-collateral)
    - [Collect Interest Payment](#collect-interest-payment)
- [Links](#links)
- [Special Shout Outs](#special-shout-outs)
 
# Smart Lending
## Introduction
Smart Lending introduces an order book model for decentralized lending for the first time. This model offers a unique twist which highlights the best traits of pooled lending and peered lending, which we believe has the potential to create the best lending experience on Cardano. Here’s a simple explanation of its key features and why it stands out.

* **Order Book Model**: Unlike typical DeFi platforms that use liquidity pools, CherryLend V2 will use an order book model. This means lenders can place loan offers, and borrowers can take loans by either using partial offers or by putting together multiple offers. For instance, if a borrower wants to take a loan of $500, the system can fulfill this loan by partially using a $1000 offer leaving the rest available for others. Or a borrower could take a $2,000 loan for which the system puts two distinct $1,000 offers together. This offer selection is done algorithmically by the platform, so it works in a completely transparent and seamless manner for the end user. The specification for the selection algorithm will be made public once it’s completely defined.

* **Flexibility in Lending and Borrowing**: This fulfillment logic adds immense flexibility. Borrowers are not constrained to accept the fixed amounts of existing loan offers. They can borrow exactly what they need, making the process more efficient and tailored to individual requirements, while eliminating the need to use single UTxOs as in pooled lending, which natively generates contention.

* **Efficiency and Accessibility**: This lending model, utilizing multiple UTxOs, inherently mitigates transaction contention, thereby dispensing with the necessity for batchers, which are commonplace in pooled lending arrangements. Also, by being built on the Cardano blockchain, CherryLend V2 benefits from low transaction fees and high transaction speeds. This makes the lending process more accessible and cost-effective for a broader range of users.

* **Innovative Approach to DeFi**: The order book model applied to lending is an innovative approach in the DeFi space. It blends elements of traditional finance (like order books) with the benefits of the eUTxO model of Cardano. This offers more competitive rates with the ability for loans to be filled based on partial or multiple existing offers, and can lead to more competitive interest rates, as lenders compete to have their funds borrowed. Additionally, this should encourage increased liquidity since loans can be partially filled, making funds more readily available.


## High-Level Overview
A lender will have 4 actions they can perform:
* Create loan offers(Does not require any smart-contract validations)
* Cancel loan offers
* Liquidate collaterals
* Collect interests
  
A borrower will have 2 actions they can perform:
* Get loan
* Repay loan


Logics of the contracts should only focus on making sure of the following things.

Lenders:
* For the amount of loan they gave out, there needs to be the correct amount of collateral locked up in the contract
and be able to liquidate it once the payment period has past
* They will collect the correct amount of interest for the loan they gave out

Borrowers: 
* They can get their collateral back once they pay back the correct amount of interest

The validators will be written in Aiken. There will be three main contracts:
* A loan contract that holds the original loan offer UTXOs, handling the validation of lenders canceling a loan offer and a borrower receiving a loan. When the borrower receives the loan, they will send their collaterals to be locked up at the collateral contract.
*  A collateral contract that will handle the validations of lenders liquidating the collateral and borrowers repaying the loan with interest. The loan and interest UTxO will be sent to the interest contract.
*  An interest contract that will handle the validations of the lenders getting back their original loan with interest 

### Splitting Loan Offer Into Multiple UTXOs And Selecting Loan UTXOs For Borrowers 
When a lender submits a loan offer, we will split the initial loan into multiple small loans to be locked up at the loan contract. Each split up UTxO will contain an equal amount of assets, but the amount may vary for different tokens. The parameters for each asset regarding the amount of asset in each loan UTxO will be disclosed openly through our GitHub repo. 

When a borrower searches for a loan, we will use an algorithm that gives priority to loan offers submitted earliest. The probability of a loan offer being selected increases exponentially with the time elapsed since the lender submitted it. Lenders that submitted a large loan will have a higher chance of being selected, as they will contribute more UTxOs to the pool.  


A loan UTxO will contain the following datum 
```
pub type LoanOfferDatum {
  loan_amount: Int,
  loan_asset: AssetClass,
  collateral_amount: Int,
  collateral_asset: AssetClass,
  interest_amount: Int,
  interest_asset: AssetClass,
  interest_percent: Int,
  loan_duration: POSIXTime,
  lender_address_hash: AddressHash,
}
```

## Smart Contract Technical Implementation

### Get Loan
##### High-Level Implementation
To validate a loan we will create a data type named ValidateLoanInfo that has the following properties
```
pub type ValidateLoanInfo {
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
We will construct ValidateLoanInfo by looping through the outputs and inputs. We will then compare the resulting two lists of ValidateLoanInfo to see if they match up.

The datum value in the UTxOs will be used to construct properties in ValidateLoanInfo when the value is not being used in the transaction and look at the actual value in the UTxO when it is being spent. For example, when calculating the amount of collateral asset being sent to the collateral script we will not be getting that value from the UTxOs datum. We will be getting that value from the assets within the UTxO. This ensures that the initial datum when the loan offers were submitted is not corrupted, and the requirements are not circumvented by changing the datum values in the outputs.

We first need to fold the input loan UTxOs and remove duplicate lender address hashes. For duplicated lender address hashes, we will sum up the collateral amount, loan amount, and interest amount, and keep the loan duration the same. We will get the collateral amount and interest amount from the datum. The loan amount will be calculated by the amount of loan assets in the input loan UTxO.

We will assume that the output collateral UTXO will be unique for each lender address. If it is not unique, the final validation will fail. We will use a filter map function to construct a list of ValidateLoanInfo. The collateral amount will be calculated by grabbing the amount of collateral assets in the output UTXO. Everything else will be constructed by looking at the datum of the collateral UTxOs. 

This is the datum for the collateral UTxO.
```
pub type CollateralDatum {
  loan_amount: Int,
  loan_asset: AssetClass,
  collateral_asset: AssetClass,
  interest_amount: Int,
  interest_asset: AssetClass,
  loan_duration: POSIXTime,
  lend_time: POSIXTime,
  lender_address_hash: AddressHash,
  total_interest_amount: Int,
  total_loan_amount: Int,
  borrower_address_hash: AddressHash,
  tx_id: TransactionId,
}
```
The filter map function will only construct a ValidateLoanInfo data from the output UTxO if the following datum are valid.
* Lend time is within the transaction validity range 
* The total loan amount is equal to the total loan amount we got from the inputs
* The total interest amount is equal to the total interest amount we got from the inputs 
* The transaction id in the datum is equal to the current transaction

##### Quick Code Walkthrough
Grab all inputs from the validator that are being spent
```
let inputs_from_script_validator: List<Input> = get_inputs_from_script()
```

Create a function that folds all the loan UTXOs from the script and returns a list of ValidateLoanInfo
```
let inputs_loan_info: List<ValidatLoanInfo> = get_inputs_loan_info()

fn get_inputs_loan_info(
  ...
) -> List<ValidateLoanInfo> {
   list.foldl(
     inputs_from_script,
     [],
     fn(input, inputs_loan) {
        ...
        when
          list.find(
            inputs_loan,
            fn(input_loan: ValidateLoanInfo) {
              ...
            },
        )
        is {
          Some(duplicate_loan) -> {
            ValidateLoanInfo {
              loan_amount: duplicate_loan.loan_amount + quantity_of(
                output.value,
                loan_offer_datum_typed.loan_asset.policy_id,
                loan_offer_datum_typed.loan_asset.asset_name,
              ),
              collateral_amount: loan_offer_datum_typed.collateral_amount + duplicate_loan.collateral_amount,
              interest_amount: loan_offer_datum_typed.interest_amount + duplicate_loan.interest_amount,
              ...
            }
          }
     }  None -> {
          let input_loan_info =
            ValidateLoanInfo {
              loan_amount: quantity_of(
                output.value,
                loan_offer_datum_typed.loan_asset.policy_id,
                loan_offer_datum_typed.loan_asset.asset_name,
              ),
              ...
            }
    }
}
```

Grab all outputs that are going to the collateral address 
```
let outputs_to_collateral_validator: List<Output> = find_script_outputs()    
```

Create a function that loops through outputs going to the collateral script and construct a list of ValidateLoanInfo
```
let outputs_collateral_info: List<ValidateLoanInfo> = get_outputs_collateral_info()

fn get_outputs_collateral_info(
  ...
) -> List<ValidateLoanInfo> {
  list.filter_map(
    script_outputs,
    fn(output: Output) {
      ...
      let output_datum_valid =
        tx_id_valid && lend_time_valid && total_loan_amount_valid && total_interest_amount_valid
      if output_datum_valid {
        Some(
          ValidateLoanInfo {
            collateral_amount: quantity_of(
              output.value,
              collateral_datum_typed.collateral_asset.policy_id,
              collateral_datum_typed.collateral_asset.asset_name,
            ),
            ...
          }
        )
      } else {
        None
      }
    }
  )
}

```

Check if the two lists of ValidateLoanInfo match up 
```
let info_difference = list.difference(input_loan_info, outputs_collateral_info)
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
  
These checks ensure that the loans are from the same transaction, the borrower is paying off the entire loan, not just parts of it, and the borrower still has time to pay off the loan.

The output interest payment UTXOs will contain datum values specifying the loan asset and repay interest asset. We will map through each UTXOs then look into the UTXO to see how much assets it contains and return a constructed ValidateInterestPayment. 

The resulting two lists of ValidateInterestPayment from the inputs and outputs will confirm that the borrower is paying the correct amount back to the lenders. 

##### Quick Code Walkthrough
Grab all input UTXOs coming from the collateral validator 
```
let inputs_from_collateral_validator: List<Input> = get_inputs_from_script()
```

Grab all output UTXOs going to the interest validator
```
let outputs_to_interest_validator: List<Output> = find_script_outputs()
```

Get the total loan repay amount and interest amount from the UTXOs going to the interest validator
```
pub type LoanAndInterestAmount {
  loan_amount: Int,
  interest_amount: Int,
}

let total_repay_amount: LoanAndInterestAmount = get_total_repay_loan_and_interest_amount()
```

Get info from the inputs 
```
let inputs_collateral_info: List<ValidateRepayInfo> = get_inputs_collateral_info()

fn get_inputs_collateral_info(
  ...
) -> List<ValidateRepayInfo> {
  list.filter_map(
    inputs_from_collateral_validator,
    fn(input_from_collateral_validator: Input) {
      ...
      let collateral_valid =
        tx_id_valid && total_loan_amount_valid && deadline_not_passed && signed_by_borrower &&                 total_interest_amount_valid
     if collateral_valid {
       Some(ValidateRepayInfo ....)
     } else {
       None
     }
    }
  )
}
```

Get the output UTxOs info 
```
let outputs_interest_info: List<ValidateRepayInfo> = get_outputs_interest_info()

fn get_outputs_interest_info(
  outputs_to_interest_validator: List<Output>,
) -> List<ValidateRepayInfo> {
  list.map(
    outputs_to_interest_validator,
    fn(output: Output) {
      ...
      let loan_and_interest_amount: LoanAndInterestAmount = get_loan_and_interest_from_output(output)
      ValidateRepayInfo {
        ...
      }
    }
  )
}
```

Check if the two lists of ValidateRepayInfo match up 
```
let info_difference = list.difference(input_loan_info, outputs_collateral_info)
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
Check the deadline has passed and the transaction is signed by the lender
```
let must_be_signed_by_lender = list.has(ctx.transaction.extra_signatories, datum.lender_address_hash)
let deadline_passed = datum.loan_duration + datum.lend_time < utils.get_lower_bound(ctx.transaction.validity_range,)
must_be_signed_by_lender && deadline_passed
```

### Collect Interest Payment
##### High Level Implementation
To cancel a loan we need to make sure the original lender is signing the transaction. The original lender's address hash is stored in the datum of the loan offer UTXO.

##### Quick Code Walkthrough
Check transaction has been signed by the original lender
```
let must_be_signed_by_lender = list.has(ctx.transaction.extra_signatories, datum.lender_address_hash)
must_be_signed_by_lender
```

### Links
[Smart Contract](https://github.com/CherryLend/cherrylend-v2-smart-contracts)

[Off-Chain](https://github.com/CherryLend/cherrylend-v2-offchain) (work in progress)


### Special Shout Outs
Special thanks to Keyan M's guidance, and open-sourced code from Anastasia Labs, Lenfi, and Sundae Labs <3


