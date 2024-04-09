<!-- markdown-toc start - Don't edit this section. Run M-x markdown-toc-refresh-toc -->
# Table of Contents

- [Smart Lending](#smart-lending)
  - [Introduction](#introduction)
  - [Design Framework](#design-framework)
  - [Technical Implementation](#technical-implementation)
    - [Grab Loan](#grab-loan)
        - [High Level](#grab-loan-high-level)
        - [Smart Contract Implementation](#grab-loan-validation)


  - [License](#license)
 
# Smart Lending

## Introduction
Smart Lending introduces an order book model for decentralized lending for the first time. This model offers a unique twist which highlights the best traits of pooled lending and peered lending, which we believe has the potential to create the best lending experience on Cardano. Here’s a simple explanation of its key features and why it stands out.

* **Order Book Model**: Unlike typical DeFi platforms that use liquidity pools, CherryLend V2 will use an order book model. This means lenders can place loan offers, and borrowers can take loans by either using partial offers or by putting together multiple offers. For instance, if a borrower wants to take a loan of $500, the system can fulfill this loan by partially using a $1000 offer leaving the rest available for others. Or a borrower could take a $2,000 loan for which the system puts two distinct $1,000 offers together. This offer selection is done algorithmically by the platform, so it works in a completely transparent and seamless manner for the end user. The specification for the selection algorithm will be made public once it’s completely defined.

* **Flexibility in Lending and Borrowing**: This fulfillment logic adds immense flexibility. Borrowers are not constrained to accept the fixed amounts of existing loan offers. They can borrow exactly what they need, making the process more efficient and tailored to individual requirements, while eliminating the need to use single UTxOs as in pooled lending, which natively generates contention.

* **Efficiency and Accessibility**: This lending model, utilizing multiple UTxOs, inherently mitigates transaction contention, thereby dispensing with the necessity for batchers, which are commonplace in pooled lending arrangements. Also, by being built on the Cardano blockchain, CherryLend V2 benefits from low transaction fees and high transaction speeds. This makes the lending process more accessible and cost-effective for a broader range of users.

* **Innovative Approach to DeFi**: The order book model applied to lending is an innovative approach in the DeFi space. It blends elements of traditional finance (like order books) with the benefits of the eUTxO model of Cardano. This offers more competitive rates with the ability for loans to be filled based on partial or multiple existing offers, and can lead to more competitive interest rates, as lenders compete to have their funds borrowed. Additionally, this should encourage increased liquidity since loans can be partially filled, making funds more readily available.


## Design Framework
**Lenders**  
* For the amount of loan they gave out, there needs to be the correct amount of collateral locked up in the contract
* They are able liquidate the collateral once the interest has not been paid by the deadline
* Only they can liquidate the collateral, cancel the loan offers

**Borrowers**  
* They can get back their collateral once they pay back the correct amount of interest



## Technical Implementation

### Grab Loan




