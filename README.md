|# |السؤال                       |الإجابة                                                                                                                   |
|--|-----------------------------|--------------------------------------------------------------------------------------------------------------------------|
|2 |Application Name             |FabMisr Open Contract Bridge Utility                                                                                      |
|3 |Description                  |Generates and transmits NI XML bridge files (Contract, Card, Card Attribute) for approved FabMisr credit card applications|
|4 |Accessibility                |N/A – background scheduled utility, no UI                                                                                 |
|11|Application Life Cycle Status|Under Planning (newly deployed)                                                                                           |
|12|Application Type             |Business                                                                                                                  |
|15|Business Department          |Cards                                                                                                                     |
|17|Business Service             |Cards                                                                                                                     |
|19|DB Instance name             |SQL Server                                                                                                                |
|23|Network Layer                |FABMISR Data Center                                                                                                       |
|25|Impact Statement             |Delay in credit card contract/activation processing with NI if unavailable                                                |
|26|License Count                |N/A (in-house developed)                                                                                                  |
|27|License Type                 |N/A                                                                                                                       |
|28|PCI Flag                     |Yes (processes card data – PAN, CVV)                                                                                      |
|29|Public/Customer Facing       |No                                                                                                                        |
|30|Service Window               |Daily batch, runs once at 12:00 AM                                                                                        |
|33|IT infra or Business App     |Business App                                                                                                              |
|36|Who is intended to use it    |Automated system process – no direct end users                                                                            |
|37|Business process             |Credit card contract creation & issuance                                                                                  |
|38|Category                     |Integration                                                                                                               |
|39|Is there a LB                |No                                                                                                                        |
|40|External dependency          |Yes – NI (TWCMS)                                                                                                          |
|41|Upstream/Downstream          |Upstream: FabMisr bank DB — Downstream: NI (TWCMS)                                                                        |
|42|APIs Used                    |None (file-based XML, transmitted via FTP)                                                                                |
|43|3rd Parties                  |NI (Network International)                                                                                                |
|44|Assumed Business Impact      |Delayed card activation, requires manual resend                                                                           |
|45|What needs to be monitored   |Daily task execution status (success/failure), generated file count                                                       |
