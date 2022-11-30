<pre>

// In this second code journal of this cluster I'm going to comment some test sections on the cw20-ics20 contract


#[cfg(test)] //test section
mod test {
    use super::*; // we import all the crates from the contract.rs file
    use crate::test_helpers::*;  // an helper file for our test function

    use cosmwasm_std::testing::{mock_env, mock_info, MOCK_CONTRACT_ADDR}; // we need these creates for unit tests, so we can mock info, enviroment, storage..
    use cosmwasm_std::{coin, coins, CosmosMsg, IbcMsg, StdError, Uint128};

    use crate::state::ChannelState; 
    use cw_utils::PaymentError; // some variants of errors for the payment functions



    // now we start with our first test
    // first we are going to test the ChannelInfo and query the list of channel
    #[test]
    fn setup_and_query() {
        let deps = setup(&["channel-3", "channel-7"], &[]); // we instantiate the contract with mock deps, info and channels

        let raw_list = query(deps.as_ref(), mock_env(), QueryMsg::ListChannels {}).unwrap();      // query the list of channels, we'll have "channel-3", "channel-7"
        let list_res: ListChannelsResponse = from_binary(&raw_list).unwrap();     // we unwrap the response from binary to a vec of ChannelInfo
        assert_eq!(2, list_res.channels.len());     //we check if the list of channels is 2 (should be 2 because we only instered "channel-3", "channel-7")
        // we check if the order in the list is correct
        assert_eq!(mock_channel_info("channel-3"), list_res.channels[0]);
        assert_eq!(mock_channel_info("channel-7"), list_res.channels[1]);

        // we query "channel-3"
        let raw_channel = query(
            deps.as_ref(),
            mock_env(),
            QueryMsg::Channel {
                id: "channel-3".to_string(),
            },
        )
        .unwrap();
        let chan_res: ChannelResponse = from_binary(&raw_channel).unwrap(); // / we unwrap the response from binary to ChannelResponse struct
        assert_eq!(chan_res.info, mock_channel_info("channel-3"));   // we check if the channel id is correct
        // total sent and balance should be 0 if not this will panic
        assert_eq!(0, chan_res.total_sent.len()); 
        assert_eq!(0, chan_res.balances.len());
        
        // now we query a non existing channel, the line 56 should trow the error
        let err = query(
            deps.as_ref(),
            mock_env(),
            QueryMsg::Channel {
                id: "channel-10".to_string(),
            },
        )
        .unwrap_err();
        assert_eq!(err, StdError::not_found("cw20_ics20::state::ChannelInfo"));
    }
    
    
    
    // now antoher test but with the transfer message
    
    #[test]
    fn proper_checks_on_execute_native() {
        let send_channel = "channel-5";   // the channel where we'll send the funds
        let mut deps = setup(&[send_channel, "channel-10"], &[]);  // we initialize again with mock info, env, ...
        
        // we create the TransferMsg
        let mut transfer = TransferMsg {
            channel: send_channel.to_string(), 
            remote_address: "foreign-address".to_string(),   // the address who receive the funds in the send_channel 
            timeout: None,    // How long the packet lives in seconds
        };

        // works with proper funds
        let msg = ExecuteMsg::Transfer(transfer.clone());    // create the ExecuteMsg:Transfer with the message we created above
        let info = mock_info("foobar", &coins(1234567, "ucosm"));    // we  mock the sender and the funds
        let res = execute(deps.as_mut(), mock_env(), info, msg).unwrap();  // we execute the msg with our mocked variables
        assert_eq!(res.messages[0].gas_limit, None);    // gas should be 0 if not this will panic
        assert_eq!(1, res.messages.len());      // we should have only 1 message in the response
        
        // we create our ibc message to check if the transfer gone well
        if let CosmosMsg::Ibc(IbcMsg::SendPacket {
            channel_id,
            data,
            timeout,
        }) = &res.messages[0].msg
        {
            let expected_timeout = mock_env().block.time.plus_seconds(DEFAULT_TIMEOUT);  
            assert_eq!(timeout, &expected_timeout.into());  // timeout should be the default one
            assert_eq!(channel_id.as_str(), send_channel);  
            let msg: Ics20Packet = from_binary(data).unwrap();
            // in the next 4 lines we check if amountm, denom, sender and receiver are equal with the one the sendend in the transfer message
            assert_eq!(msg.amount, Uint128::new(1234567));
            assert_eq!(msg.denom.as_str(), "ucosm");
            assert_eq!(msg.sender.as_str(), "foobar");
            assert_eq!(msg.receiver.as_str(), "foreign-address");
        } else {
            panic!("Unexpected return message: {:?}", res.messages[0]);  // transfer gone wrong, we panick the contract
        }

        // we verify the case when the sender don't sent funds
        let msg = ExecuteMsg::Transfer(transfer.clone());
        let info = mock_info("foobar", &[]);
        let err = execute(deps.as_mut(), mock_env(), info, msg).unwrap_err();
        assert_eq!(err, ContractError::Payment(PaymentError::NoFunds {}));  // should trow this error

        //  we verify the case when send multiple tokens funds
        let msg = ExecuteMsg::Transfer(transfer.clone());
        let info = mock_info("foobar", &[coin(1234567, "ucosm"), coin(54321, "uatom")]);
        let err = execute(deps.as_mut(), mock_env(), info, msg).unwrap_err();
        assert_eq!(err, ContractError::Payment(PaymentError::MultipleDenoms {}));   // should trow this error

        // we verify the case with bad channel id
        transfer.channel = "channel-45".to_string();
        let msg = ExecuteMsg::Transfer(transfer);
        let info = mock_info("foobar", &coins(1234567, "ucosm"));
        let err = execute(deps.as_mut(), mock_env(), info, msg).unwrap_err();
        assert_eq!(
            err,
            ContractError::NoSuchChannel {
                id: "channel-45".to_string()
            }
        );  // should trow this error
    }



</pre>
