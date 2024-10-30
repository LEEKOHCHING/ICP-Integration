# ICP-Integration
How to using vb.net & Bash Script to monitoring ICP Coin &amp; ckBTC

Our team lacks proficiency in RUST, as well as Node.js and JavaScript. Instead, we accomplished integration using our familiar VB.NET and Bash scripts. If readers encounter similar issues, they can refer to the following approach. However, developers with expertise in RUST or Node.js are advised against using this method.

```vb.net
  <WebMethod()>
    Public Sub ICPGetTask()

        Dim gCnnPCE As SqlConnection
        gCnnPCE = New SqlConnection(ConnStr)
        Try
            gCnnPCE.Open()
            Dim str As String
            str = "SELECT TOP 1 * FROM TxReceiveCoin WHERE COIN = @COIN AND Status = 0 ORDER BY ID DESC"
            Dim cmd As New SqlCommand(str, gCnnPCE)
            cmd.Parameters.AddWithValue("@COIN", "ICP")
            Dim da As New SqlDataAdapter(cmd)
            Dim dt As New DataTable
            da.Fill(dt)

            If dt.Rows.Count = 0 Then
                Context.Response.Write("0")
            Else
                Context.Response.Write(dt.Rows(0).Item("Address").ToString)
            End If


        Catch ex As Exception
            WriteErrror("ICPGetTask", ex.Message)
        Finally
            gCnnPCE.Close()
            gCnnPCE.Dispose()
        End Try

    End Sub
```

We first create an API for TabbyPOS, which is responsible for receiving requests from POS machines and monitoring a specific ICP address. Once a new request is received, the API will respond with 1. When the Linux Ubuntu server receives this response, it executes the following commands to check the balance of the ICP address.

```bash
#!/bin/bash
while true; do
    response=$(curl [API URL]/ICPGetTask)
    if [ "$response" -eq 0 ]; then
       echo "Response is equal to 0"
    else
         # Run the command if response is not zero
        output=$(dfx ledger --network ic balance $(dfx ledger account-id --of-principal "$response"))
        encoded_output=$(urlencode "$output")
        api_url="[API URL]/ICPUpdateTask?Address=$response&Balance=$encoded_output"
        curl "$api_url"       
    fi
    # Sleep for 2 seconds
    sleep 2
done
```

Once we obtain the ICP amount for a specific address, we will check in the background whether the order has been successfully paid.

Due to security and confidentiality reasons, we have not disclosed the code for the function that determines whether the payment was successful.

![demo](https://github.com/LEEKOHCHING/ICP-Integration/blob/main/ezgif-2-99f7e52fc4.gif)

# ckBTC-Integration
The next step is integrating ckBTC. The overall concept is quite similar, with the main difference being that ckBTC is an ICRC-2-compliant token, which makes the balance query process different. Below is the code for querying ckBTC balance.
```bash
#!/bin/bash
while true; do
    response=$(curl [API URL]/ckBTCGetTask)
    if [ "$response" -eq 0 ]; then
       echo "Response is equal to 0"
    else
         # Run the command if response is 1     
         output=$(dfx canister --network ic call mxzaz-hqaaa-aaaar-qaada-cai icrc1_balance_of "(record { owner = principal \""$response"\"; subaccount = null; })")
   encoded_output=$(urlencode "$output")
        api_url="https://www.apikolmenpro.com/wsTabby/tabby.asmx/ckBTCUpdateTask?Address=$response&Balance=$encoded_output"
        curl "$api_url"    
    fi
    # Sleep for 3 seconds
    sleep 3
done
```

![demo](https://github.com/LEEKOHCHING/ICP-Integration/blob/main/ezgif-3-b47604cbb8.gif)

TabbyPOS will use ckBTC as membership reward points for purchases, so it will involve how to send ckBTC.
```bash
#!/bin/bash
while true; do
    response=$(curl [API URL]/ckBTCGetTaskSendBTC)
    if [ "$response" -eq 0 ]; then
       echo "Response is equal to 0"
    else
	IFS=','read -r address amount <<< "$response"
	echo"Address: $address"
	echo"Amount: $amount"
         # Run the command if response is 1     
         output=$(dfx canister --network ic call mxzaz-hqaaa-aaaar-qaada-cai icrc1_transfer "(record { from_subaccount = null; to = record { owner = principal \""$address"\"; subaccount = null }; amount = $amount; })")
   encoded_output=$(urlencode "$output")
        api_url="https://www.apikolmenpro.com/wsTabby/tabby.asmx/ckBTCUpdateTaskSendBTC?Address=$response&Remarks=$encoded_output"
        curl "$api_url"    
    fi
    # Sleep for 3 seconds
    sleep 3
done
```

