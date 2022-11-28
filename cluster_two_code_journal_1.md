<pre>
// In this code journal I'm going to rewiew some code from the cw20-ics20, an IBC enable contract that allows us to send CW20 tokens from one chain over 
// the standard ICS20 protocol to the bank module of another chain.

// In this first code journal for this cluster I'm going to take a look at the cw20-ics20/src/contract.rs file, since it'll help me a lot for my capstone.
// In particolar I'm going to take a look at the instantiate, execute and query entry_point


#[cfg(not(feature = "library"))]
use cosmwasm_std::entry_point;
use cosmwasm_std::{
    from_binary, to_binary, Addr, Binary, Deps, DepsMut, Env, IbcMsg, IbcQuery, MessageInfo, Order,
    PortIdResponse, Response, StdError, StdResult,
};
use semver::Version;      // To use the Semantic Versioning, a "tool" used to help with checking version numbers

use cw2::{get_contract_version, set_contract_version};    // we need to get the previous contract version for migration
use cw20::{Cw20Coin, Cw20ReceiveMsg}; 
use cw_storage_plus::Bound;     // is used to define the two ends of a range

use crate::amount::Amount;
use crate::error::ContractError;
use crate::ibc::Ics20Packet; // It's the format to send ics20 packets
use crate::migrations::{v1, v2}; // Not really sure about that
// now we import all the type of messages we'll need
use crate::msg::{
    AllowMsg, AllowedInfo, AllowedResponse, ChannelResponse, ConfigResponse, ExecuteMsg, InitMsg,
    ListAllowedResponse, ListChannelsResponse, MigrateMsg, PortResponse, QueryMsg, TransferMsg,
};
use crate::state::{
    increase_channel_balance, AllowInfo, Config, ADMIN, ALLOW_LIST, CHANNEL_INFO, CHANNEL_STATE,
    CONFIG,
};
use cw_utils::{maybe_addr, nonpayable, one_coin}; // first is used for pagination, other 2 for checking the MessageInfo funds

// Till here we imported the crates we need

// version info for migration info
const CONTRACT_NAME: &str = "crates.io:cw20-ics20";
const CONTRACT_VERSION: &str = env!("CARGO_PKG_VERSION");

// Let's take a look in the instantiate entry point:
#[cfg_attr(not(feature = "library"), entry_point)]
pub fn instantiate(
    mut deps: DepsMut,
    _env: Env,
    _info: MessageInfo,
    msg: InitMsg,
) -> Result<Response, ContractError> {
    set_contract_version(deps.storage, CONTRACT_NAME, CONTRACT_VERSION)?; // we store the original version, so we can migrate it
    // we declare our Config with the default timeout and the gas limit
    let cfg = Config {
        default_timeout: msg.default_timeout,
        default_gas_limit: msg.default_gas_limit,
    };
    CONFIG.save(deps.storage, &cfg)?; // we store the config on chain 

    let admin = deps.api.addr_validate(&msg.gov_contract)?; // we check if msg.gov_contract is a valid address and assign it to admin
    ADMIN.set(deps.branch(), Some(admin))?; // we save the admin

    // add all allows
    for allowed in msg.allowlist {
        let contract = deps.api.addr_validate(&allowed.contract)?;
        let info = AllowInfo {
            gas_limit: allowed.gas_limit,
        };
        ALLOW_LIST.save(deps.storage, &contract, &info)?;
    }
    Ok(Response::default())
}


// Now the execute:

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn execute(
    deps: DepsMut,
    env: Env,
    info: MessageInfo,
    msg: ExecuteMsg,
) -> Result<Response, ContractError> {
    match msg {
        ExecuteMsg::Receive(msg) => execute_receive(deps, env, info, msg), // Receive messages from a cw20 contract
        ExecuteMsg::Transfer(msg) => {                                     // This allows us to transfer 1 native token
            let coin = one_coin(&info)?;                                   
            execute_transfer(deps, env, msg, Amount::Native(coin), info.sender)
        }
        ExecuteMsg::Allow(allow) => execute_allow(deps, env, info, allow), // This allow a new cw20 token to be sent (should be called only by gov_contract)
        ExecuteMsg::UpdateAdmin { admin } => {                             // If we want to update the admin (called only by current admin)
            let admin = deps.api.addr_validate(&admin)?;
            Ok(ADMIN.execute_update_admin(deps, info, Some(admin))?)
        }
    }
}



// And the last, query:

#[cfg_attr(not(feature = "library"), entry_point)]
pub fn query(deps: Deps, _env: Env, msg: QueryMsg) -> StdResult<Binary> {
    match msg {
        QueryMsg::Port {} => to_binary(&query_port(deps)?),               // we returns the port ID this contract has bound, so you can create channels. 
        QueryMsg::ListChannels {} => to_binary(&query_list(deps)?),       // that returns a list of all channels that have been created on this contract.
        QueryMsg::Channel { id } => to_binary(&query_channel(deps, id)?), // returns more detailed information on one specific channel like the current balance  or the total amount that has ever been sent on a specif channel.
        QueryMsg::Config {} => to_binary(&query_config(deps)?),           // simply show the Config
        QueryMsg::Allowed { contract } => to_binary(&query_allowed(deps, contract)?),   // query if a given cw20 contract is allowed
        QueryMsg::ListAllowed { start_after, limit } => {                 // list of all allowed cw20 contracts
            to_binary(&list_allowed(deps, start_after, limit)?)
        }
        QueryMsg::Admin {} => to_binary(&ADMIN.query_admin(deps)?),       // show the admin
    }
}








</pre>
