この構成図は、VPC内のEC2からAWS PrivateLink (VPC Endpoint) を経由して、Amazon Bedrockにプライベートに接続する構成を示しています。
```mermaid
graph TD
    subgraph AWSCloud ["AWS Cloud"]
        subgraph VPC ["VPC (10.0.0.0/16)"]
            subgraph AZ_A ["Availability Zone A"]
                subgraph Subnet_A ["Private Subnet A"]
                    EC2_A["EC2 Instance A"]
                end
            end

            subgraph AZ_B ["Availability Zone B"]
                subgraph Subnet_B ["Private Subnet B"]
                    EC2_B["EC2 Instance B"]
                end
            end

            subgraph Subnet_Shared ["Private Subnet (Interface)"]
                VPCE["VPC Endpoint<br>(Bedrock Runtime)"]
            end
        end

        subgraph AWS_Services ["AWS Services (Public/Managed)"]
            Bedrock(("Amazon Bedrock"))
        end
    end

    %% Connections
    EC2_A -->|Private Connection| VPCE
    EC2_B -->|Private Connection| VPCE
    VPCE -->|AWS Backbone| Bedrock

    %% Styling
    style AWSCloud fill:#f9f9f9,stroke:#333,stroke-width:2px
    style VPC fill:#e1f5fe,stroke:#01579b,stroke-width:2px
    style Subnet_A fill:#ffffff,stroke:#81d4fa,stroke-dasharray: 5 5
    style Subnet_B fill:#ffffff,stroke:#81d4fa,stroke-dasharray: 5 5
    style Bedrock fill:#ff9900,stroke:#fff,color:#fff
    style VPCE fill:#4caf50,stroke:#fff,color:#fff
```

この構成図は、Interface VPC Endpointを経由してPrivate API Gatewayにアクセスし、そこからVPC LinkおよびInternal Load Balancer（ALB）を通じてEC2インスタンスへ通信する、完全にクローズドなネットワーク構成を示しています。

## 構成図 (Mermaid)

```mermaid
graph TD
    subgraph External ["External / Other VPC"]
        Client["Client Application"]
    end

    subgraph AWSCloud ["AWS Cloud"]
        subgraph AWS_Services ["AWS Services (Managed)"]
            APIGW[["Private API Gateway"]]
            VPCLink["VPC Link"]
        end

        subgraph VPC ["VPC (10.0.0.0/16)"]
            subgraph Endpoints ["Endpoint Subnet"]
                VPCE["VPC Endpoint<br>(execute-api)"]
            end

            ILB(["Internal Load Balancer<br>(ALB)"])

            subgraph AZ_A ["Availability Zone A"]
                subgraph Subnet_A ["Private Subnet A"]
                    EC2_A["EC2 Instance A"]
                end
            end

            subgraph AZ_B ["Availability Zone B"]
                subgraph Subnet_B ["Private Subnet B"]
                    EC2_B["EC2 Instance B"]
                end
            end
        end
    end

    %% Connections
    Client -->|HTTPS/Private DNS| VPCE
    VPCE -.->|AWS Backbone| APIGW
    APIGW -->|Route via| VPCLink
    VPCLink -.->|ENI Connection| ILB
    ILB -->|Forward| EC2_A
    ILB -->|Forward| EC2_B

    %% Styling
    style AWSCloud fill:#f9f9f9,stroke:#333,stroke-width:2px
    style VPC fill:#fff9c4,stroke:#fbc02d,stroke-width:2px
    style Subnet_A fill:#ffffff,stroke:#81d4fa,stroke-dasharray: 5 5
    style Subnet_B fill:#ffffff,stroke:#81d4fa,stroke-dasharray: 5 5
    style APIGW fill:#880e4f,stroke:#fff,color:#fff
    style VPCE fill:#4caf50,stroke:#fff,color:#fff
    style ILB fill:#0288d1,stroke:#fff,color:#fff
    style VPCLink fill:#7e57c2,stroke:#fff,color:#fff
```
