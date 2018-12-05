# Hyperledger Fabric Capability Configuration
This project demonstrate how to configure **Capability** on a Hyperledger Fabric network. Specifically we want to use **Capability** feature in a **System Chaincode**. 

We will use the [Simple Network](https://github.com/chainforce/native-fabric) project and modifying its default capabilities to insert a capability for the system chaincode **example02**.


## Create Capability Transaction
The general instruction is provided by Fabric [Channel Update](https://hyperledger-fabric.readthedocs.io/en/latest/channel_update_tutorial.html) and [Capability Specification](https://hyperledger-fabric.readthedocs.io/en/latest/capability_requirements.html). It is recommended to get the existing configuration from the network; modify what you need, and submit a configuration transaction with the modified data. The specific steps to create and send a configuration transaction for modifying **Capability** is given below:

1. Fetch existing config from the channel, `mych`, and store it in `config_block.pb`. This is a binary file containing the protobuf data of the most recent configuration block on the channel. Note the `FABBIN` and `FABCONF` from the [Simple Network](https://github.com/chainforce/native-fabric) project.
```
FABRIC_CFG_PATH=$FABCONF $FABBIN/peer channel fetch config config_block.pb -o 127.0.0.1:7050 -c mych
```

2. Translate config to json from protobuf so that we can edit. The output file is `config.json`. This step creates a temporary file `tmpa`, which we will remove at the end. Note that you may be able to pipe the output directly without using `tmpa`, but it doesn't work on my machine (eg configtxlator proto_decode --input config_block.pb --type common.Block | jq .data.data[0].payload.data.config > config.json).

```
$FABBIN/configtxlator proto_decode --input config_block.pb --type common.Block --output=tmpa; jq .data.data[0].payload.data.config tmpa > config.json
```

3. Modify `config.json` to add **capability configuration** and save the content to `modified_config.json`. We add a **capability** for our sample system chaincode called `example02`. The path is `Application.values.Capabilities`.
```
          "Capabilities": {
            "mod_policy": "Admins",
            "value": {
              "capabilities": {
                "V1_3": {},
                "example02": {}
              }
            },
```

At this point, we have a modified configuration in json, but we need to figure out the delta changes so that we can create a configuration transaction. To do this, we first translate the original and modified json files into protobuf then calculate the delta. It is done in steps 4, 5, and 6 below.

4. Create protobuf from the original json file `config.json`
```
$FABBIN/configtxlator proto_encode --input config.json --type common.Config --output config.pb
```

5. Create protobuf from the modified json file `modified_config.json`
```
$FABBIN/configtxlator proto_encode --input modified_config.json --type common.Config --output modified_config.pb
```

6. Create the update delta from `config.pb` and `modified_config.pb`
```
$FABBIN/configtxlator compute_update --channel_id mych --original config.pb --updated modified_config.pb --output cap_update.pb
```

7. Translate the update delta `cap_update.pb` to json. This step is necessary to add the transaction envelope to the update data in step 8.
```
$FABBIN/configtxlator proto_decode --input cap_update.pb --type common.ConfigUpdate --output tmpb; jq . tmpb > cap_update.json
```

8. Add envelope wrapper to the update delta
```
echo '{"payload":{"header":{"channel_header":{"channel_id":"mych", "type":2}},"data":{"config_update":'$(cat cap_update.json)'}}}' > tmpc; jq . tmpc > cap_update_in_envelope.json
```

9. Finally, convert the json file to protobuf `cap_update_in_envelope.pb`. This is the configuration transaction binary that we can send to the network.
```
$FABBIN/configtxlator proto_encode --input cap_update_in_envelope.json --type common.Envelope --output cap_update_in_envelope.pb
```

10. Send update configuration transaction
```
FABRIC_CFG_PATH=$FABCONF $FABBIN/peer channel update -f cap_update_in_envelope.pb -c mych -o 127.0.0.1:7050
```

The **peer** will exit at this point as the capability validation rejects the newly added capability `example02`. We need to further modify Fabric validation code to allow **System Chaincode** to use **Capability** feature

### caps validation stacktrace
github.com/hyperledger/fabric/core/peer.capabilitiesSupportedOrPanic(0x4df5b20, 0xc423417a00)
	/mywork/obc/src/github.com/hyperledger/fabric/core/peer/peer.go:139 +0x311
github.com/hyperledger/fabric/core/peer.(*chainSupport).Apply(0xc421ee3f00, 0xc422133200, 0x0, 0x0)
	/mywork/obc/src/github.com/hyperledger/fabric/core/peer/peer.go:125 +0x155
github.com/hyperledger/fabric/core/committer/txvalidator.(*TxValidator).validateTx(0xc421ff7b90, 0xc421e1df70, 0xc4220f04e0)
	/mywork/obc/src/github.com/hyperledger/fabric/core/committer/txvalidator/validator.go:424 +0x826
github.com/hyperledger/fabric/core/committer/txvalidator.(*TxValidator).Validate.func1.1(0xc421ff7b90, 0xc4220745c0, 0xc4220f04e0, 0x0, 0xc423334000, 0x34f8, 0x3500)
	/mywork/obc/src/github.com/hyperledger/fabric/core/committer/txvalidator/validator.go:158 +0xcb
created by github.com/hyperledger/fabric/core/committer/txvalidator.(*TxValidator).Validate.func1
	/mywork/obc/src/github.com/hyperledger/fabric/core/committer/txvalidator/validator.go:155 +0x109

core/committer/txvalidator/validator #55 Capabilities() channelconfig.ApplicationCapabilities
