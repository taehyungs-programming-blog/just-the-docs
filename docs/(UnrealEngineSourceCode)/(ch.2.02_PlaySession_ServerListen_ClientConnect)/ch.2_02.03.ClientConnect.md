---
layout: default
title: "02-03. ClientConnect"
parent: "([Network] 02.PlaySession, ServerListen, ClientConnect)"
grand_parent: "(UE SourceCode 분석 🤖)"
nav_order: 1
---

## Table of contents
{: .no_toc .text-delta }

1. TOC
{:toc}

---

```cpp
/** called to actually start the game when doing play/simulate in editor */
virtual FGameInstancePIEResult StartPlayInEditorGameInstance(ULocalPlayer* LocalPlayer, const FGameInstancePIEParameters& Params)
{
    UEditorEngine* const EditorEngine = CastChecked<UEditorEngine>(GetEngine());
    {
        // for clients, just connect to the server
        // for networking, we add this:
        // - in networking, client-side and server(or standalone) game has different path
        if (Params.NetMode == PIE_Client)
        {
            // in "Networking - ServerListen", skip this part, we'll cover this in "ClientConnect"

            // << 여기서 부터 시작이다. >>

            // BaseURL has (127.0.0.1:1777)
            FURL BaseURL = WorldContext->LastURL;
            FString URLString(TEXT("127.0.0.1"));

            // we set our server port as 17777, which will be overriden
            uint16 ServerPort = 0;
            if (Params.EditorPlaySettings->GetServerPort(ServerPort))
            {
                URLString += FString::Printf(TEXT(":%hu"), ServerPort);
            }

            // before getting into UEngine::Browse, see rest of code below:
            // - we are waiting to complete connect process
            // - see UEngine::Browse (goto 001: ClientConnect)
            if (EditorEngine->Browse(*WorldContext, FURL(&BaseURL, *URLString, (ETravelType)TRAVEL_Absolute), Error) == EBrowseReturnVal::Pending)
            {
                EditorEngine->TransitionType = ETransitionType::WaitingToConnect;
            }
            else
            {
                return FGameInstancePIEResult::Failure();
            }
        }
        else 
        {
            // ...
        }
    }
}
```

```cpp
/** browse to a specified URL, relative to the current one */
// we can think of 'Browse' as browsing session game provided by server, so first connect to the server
virtual EBrowseReturnVal::Type Browse(FWorldContext& WorldContext, FURL URL, FString& Error)
{
    //...

    if (URL.IsLocalInternal())
    {
        //...
    }
    else if (URL.IsInternal() && GIsClient)
    {
        // we are create UPendingNetGame:
        // - server has infromation which map we have to load
        // - to get map information, first we need to connect the server
        // - before connecting to server, we create UPendingNetGame, temporary stage
        //   - to connect the server, we need a NetDriver to communicate
        
        //   - note that we are putting new instance of UPendingNetGame to WorldContext's PendingNetGame (*** see FWorldContext::PendingNetGame)
        // - see UPendingNetGame::Initialize briefly
        //   - before looking into InitNetDriver(), see that we'll return EBrowseReturnVal::Pending:
        //   - this is because, we don't know which map do we have to load!
        
        WorldContext.PendingNetGame = NewObject<UPendingNetGame>();
        WorldContext.PendingNetGame->Initialize(URL);
        WorldContext.PendingNetGame->InitNetDriver();

        return EBrowseReturnVal::Pending;
    }
}
```

```cpp
// note that UPendingNetGame inherits from FNetworkNotify which is similar to UWorld
    // 잠깐 사용할 NetGame이라 생각하자.
class UPendingNetGame : public UObject, FNetworkNotify
{
```

```cpp
void InitNetDriver()
{
    // try to create NetDriver
    // we create client IpNetDriver with UPendingNetGame
    // NAME_PendingNetDriver == "PendingNetDriver"
    // NAME_GameNetDriver == "GameNetDriver"
    // - CreateNamedNetDriver pass UPendingNetGame not UWorld (different from server)

    if (GEngine->CreateNamedNetDriver(this, NAME_PendingNetDriver, NAME_GameNetDriver))
    {
        NetDriver = GEngine->FindNamedNetDriver(this, NAME_PendingNetDriver);
    }

    // see IpNetDriver::InitConnect (goto 005: ClientConnect)
    if (NetDriver->InitConnect(this, URL, ConnectionError))
    {
        // @todo
    }
}
```

```cpp
virtual bool InitConnect(FNetworkNotify* InNotify, const FURL& ConnectURL, FString& Error) override
{
    // we pass "bInitAsClient == true": (in server-side code, we pass it as 'false')
    // - see UIpNetDriver::InitBase caring about bInitAsClient == ture, 
        // 소켓을 만드는 과정이고 bInitAsClient == ture임을 기억하자
        // Server에서 한 번 다룸 ㅎㅎ..
    if (!InitBase(true, InNotify, ConnectURL, false, Error))
    {
        return false;
    }

    // create new connection
    // NetConnectionClass is 'IpNetConnection'
    // - see UNetDriver::ServerConnection (***)
    //   - note that NetConnectionClass is loaded in InitConnectionClass()
    ServerConnection = NewObject<UNetConnection>(GetTrasientPackage(), NetConnectionClass);

    ServerConnection->InitLocalConnection(this, SocketPrivate.Get(), ConnectURL, USOCK_Pending);
    
    // @todo
}
```