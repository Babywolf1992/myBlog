---
layout: post
title:  ZYQAssetPickerController奇怪的奔溃问题
date:   2017-08-07 23:20:00 +0800
tags: 能工巧匠集
---

最近公司项目开发IM系统，使用到的openfire服务，对应的客户端使用XMPP协议。这一篇文章，主要讲一下在iOS端XMPP的连接流程。

首先是这样两种连接，根据不同的情况进行选择。我们使用的是C2S模式。

    ////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
    #pragma mark C2S Connection
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
    
    - (BOOL)connectToHost:(NSString *)host onPort:(UInt16)port withTimeout:(NSTimeInterval)timeout error:(NSError **)errPtr
    {
    	NSAssert(dispatch_get_specific(xmppQueueTag), @"Invoked on incorrect queue");
    	
    	XMPPLogTrace();
    	
    	BOOL result = [asyncSocket connectToHost:host onPort:port error:errPtr];
    	
    	if (result && [self resetByteCountPerConnection])
    	{
    		numberOfBytesSent = 0;
    		numberOfBytesReceived = 0;
    	}
    	
    	if(result)
        {
            [self startConnectTimeout:timeout];
        }
    	
    	return result;
    }
    
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
    #pragma mark P2P Connection
    ////////////////////////////////////////////////////////////////////////////////////////////////////////////////////////
    
    /**
     * Starts a P2P connection to the given user and given address.
     * This method only works with XMPPStream objects created using the initP2P method.
     * 
     * The given address is specified as a sockaddr structure wrapped in a NSData object.
     * For example, a NSData object returned from NSNetservice's addresses method.
    **/
    - (BOOL)connectTo:(XMPPJID *)jid withAddress:(NSData *)remoteAddr withTimeout:(NSTimeInterval)timeout error:(NSError **)errPtr
    {
    	XMPPLogTrace();
    	
    	__block BOOL result = YES;
    	__block NSError *err = nil;
    	
    	dispatch_block_t block = ^{ @autoreleasepool {
    		
    		if (state != STATE_XMPP_DISCONNECTED)
    		{
    			NSString *errMsg = @"Attempting to connect while already connected or connecting.";
    			NSDictionary *info = @{NSLocalizedDescriptionKey : errMsg};
    			
    			err = [NSError errorWithDomain:XMPPStreamErrorDomain code:XMPPStreamInvalidState userInfo:info];
    			
    			result = NO;
    			return_from_block;
    		}
    		
    		if (![self isP2P])
    		{
    			NSString *errMsg = @"Non P2P streams must use the connect: method";
    			NSDictionary *info = @{NSLocalizedDescriptionKey : errMsg};
    			
    			err = [NSError errorWithDomain:XMPPStreamErrorDomain code:XMPPStreamInvalidType userInfo:info];
    			
    			result = NO;
    			return_from_block;
    		}
    		
    		// Turn on P2P initiator flag
    		flags |= kP2PInitiator;
    		
    		// Store remoteJID
    		remoteJID = [jid copy];
    		
    		NSAssert((asyncSocket == nil), @"Forgot to release the previous asyncSocket instance.");
    
    		// Notify delegates
    		[multicastDelegate xmppStreamWillConnect:self];
    
    		// Update state
    		state = STATE_XMPP_CONNECTING;
    		
    		// Initailize socket
    		asyncSocket = [[GCDAsyncSocket alloc] initWithDelegate:self delegateQueue:xmppQueue];
    		
    		NSError *connectErr = nil;
    		result = [asyncSocket connectToAddress:remoteAddr error:&connectErr];
    		
    		if (result == NO)
    		{
    			err = connectErr;
    			state = STATE_XMPP_DISCONNECTED;
    		}
    		else if ([self resetByteCountPerConnection])
    		{
    			numberOfBytesSent = 0;
    			numberOfBytesReceived = 0;
    		}
            
            if(result)
            {
                [self startConnectTimeout:timeout];
            }
    	}};
    	
    	if (dispatch_get_specific(xmppQueueTag))
    		block();
    	else
    		dispatch_sync(xmppQueue, block);
    	
    	if (errPtr)
    		*errPtr = err;
    	
    	return result;
    }

下面是GCDAsyncSocket连接的两个回调函数，至于GCDAsyncSocket是什么，我只能简单的回答一下，是iOS 中的一个开源的socket连接库。不懂的同学可以查阅一下相关资料。

    /**
     * Called when a socket connects and is ready for reading and writing. "host" will be an IP address, not a DNS name.
    **/
    - (void)socket:(GCDAsyncSocket *)sock didConnectToHost:(NSString *)host port:(UInt16)port
    {
    	// This method is invoked on the xmppQueue.
    	// 
    	// The TCP connection is now established.
    	
    	XMPPLogTrace();
        
        [self endConnectTimeout];
    	
    	#if TARGET_OS_IPHONE
    	{
    		if (self.enableBackgroundingOnSocket)
    		{
    			__block BOOL result;
    			
    			[asyncSocket performBlock:^{
    				result = [asyncSocket enableBackgroundingOnSocket];
    			}];
    			
    			if (result)
    				XMPPLogVerbose(@"%@: Enabled backgrounding on socket", THIS_FILE);
    			else
    				XMPPLogError(@"%@: Error enabling backgrounding on socket!", THIS_FILE);
    		}
    	}
    	#endif
    	
    	[multicastDelegate xmppStream:self socketDidConnect:sock];
    	
    	srvResolver = nil;
    	srvResults = nil;
    	
    	// Are we using old-style SSL? (Not the upgrade to TLS technique specified in the XMPP RFC)
    	if ([self isSecure])
    	{
    		// The connection must be secured immediately (just like with HTTPS)
    		[self startTLS];
    	}
    	else
    	{
    		[self startNegotiation];
    	}
    }
    
    /**
     * Called when a socket disconnects with or without error.
    **/
    - (void)socketDidDisconnect:(GCDAsyncSocket *)sock withError:(NSError *)err
    {
    	// This method is invoked on the xmppQueue.
    	
    	XMPPLogTrace();
        
        [self endConnectTimeout];
    	
    	if (srvResults && (++srvResultsIndex < [srvResults count]))
    	{
    		[self tryNextSrvResult];
    	}
    	else
    	{
    		// Update state
    		state = STATE_XMPP_DISCONNECTED;
    		
    		// Release the parser (to free underlying resources)
    		[parser setDelegate:nil delegateQueue:NULL];
    		parser = nil;
    		
    		// Clear any saved authentication information
    		auth = nil;
    		
    		authenticationDate = nil;
    		
    		// Clear stored elements
    		myJID_setByServer = nil;
    		myPresence = nil;
    		rootElement = nil;
    		
    		// Stop the keep alive timer
    		if (keepAliveTimer)
    		{
    			dispatch_source_cancel(keepAliveTimer);
    			keepAliveTimer = NULL;
    		}
    		
    		// Clear srv results
    		srvResolver = nil;
    		srvResults = nil;
            
            // Stop tracking IDs
            [idTracker removeAllIDs];
    		
    		// Clear any pending receipts
    		for (XMPPElementReceipt *receipt in receipts)
    		{
    			[receipt signalFailure];
    		}
    		[receipts removeAllObjects];
    		
    		// Clear flags
    		flags = 0;
    		
    		// Notify delegate
    		
    		if (parserError || otherError)
    		{
    			NSError *error = parserError ? : otherError;
    			
    			[multicastDelegate xmppStreamDidDisconnect:self withError:error];
    			
    			parserError = nil;
    			otherError = nil;
    		}
    		else
    		{
    			[multicastDelegate xmppStreamDidDisconnect:self withError:err];
    		}
    	}
    }

XMPP使用GCDAsyncSocket进行socket连接，该连接为阻塞式连接。

    int result = connect(socketFD, (const struct sockaddr *)[address bytes], (socklen_t)[address length]);
    		
    		__strong GCDAsyncSocket *strongSelf = weakSelf;
    		if (strongSelf == nil) return_from_block;
    		
    		if (result == 0)
    		{
    			dispatch_async(strongSelf->socketQueue, ^{ @autoreleasepool {
    				
    				[strongSelf didConnect:aStateIndex];
    			}});
    		}
    		else
    		{
    			NSError *error = [strongSelf errnoErrorWithReason:@"Error in connect() function"];
    			
    			dispatch_async(strongSelf->socketQueue, ^{ @autoreleasepool {
    				
    				[strongSelf didNotConnect:aStateIndex error:error];
    			}});
    		}

关于这个连接回调方法

    - (void)didConnect:(int)aStateIndex
    {
    	LogTrace();
    	
    	NSAssert(dispatch_get_specific(IsOnSocketQueueOrTargetQueueKey), @"Must be dispatched on socketQueue");
    	
    	
    	if (aStateIndex != stateIndex)
    	{
    		LogInfo(@"Ignoring didConnect, already disconnected");
    		
    		// The connect operation has been cancelled.
    		// That is, socket was disconnected, or connection has already timed out.
    		return;
    	}
    	
    	flags |= kConnected;
    	
    	[self endConnectTimeout];
    	
    	#if TARGET_OS_IPHONE
    	// The endConnectTimeout method executed above incremented the stateIndex.
    	aStateIndex = stateIndex;
    	#endif
    	
    	// Setup read/write streams (as workaround for specific shortcomings in the iOS platform)
    	// 
    	// Note:
    	// There may be configuration options that must be set by the delegate before opening the streams.
    	// The primary example is the kCFStreamNetworkServiceTypeVoIP flag, which only works on an unopened stream.
    	// 
    	// Thus we wait until after the socket:didConnectToHost:port: delegate method has completed.
    	// This gives the delegate time to properly configure the streams if needed.
    	
    	dispatch_block_t SetupStreamsPart1 = ^{
    		#if TARGET_OS_IPHONE
    		
    		if (![self createReadAndWriteStream])
    		{
    			[self closeWithError:[self otherError:@"Error creating CFStreams"]];
    			return;
    		}
    		
    		if (![self registerForStreamCallbacksIncludingReadWrite:NO])
    		{
    			[self closeWithError:[self otherError:@"Error in CFStreamSetClient"]];
    			return;
    		}
    		
    		#endif
    	};
    	dispatch_block_t SetupStreamsPart2 = ^{
    		#if TARGET_OS_IPHONE
    		
    		if (aStateIndex != stateIndex)
    		{
    			// The socket has been disconnected.
    			return;
    		}
    		
    		if (![self addStreamsToRunLoop])
    		{
    			[self closeWithError:[self otherError:@"Error in CFStreamScheduleWithRunLoop"]];
    			return;
    		}
    		
    		if (![self openStreams])
    		{
    			[self closeWithError:[self otherError:@"Error creating CFStreams"]];
    			return;
    		}
    		
    		#endif
    	};
    	
    	// Notify delegate
    	
    	NSString *host = [self connectedHost];
    	uint16_t port = [self connectedPort];
    	NSURL *url = [self connectedUrl];
    	
    	__strong id theDelegate = delegate;
    
    	if (delegateQueue && host != nil && [theDelegate respondsToSelector:@selector(socket:didConnectToHost:port:)])
    	{
    		SetupStreamsPart1();
    		
    		dispatch_async(delegateQueue, ^{ @autoreleasepool {
    			
    			[theDelegate socket:self didConnectToHost:host port:port];
    			
    			dispatch_async(socketQueue, ^{ @autoreleasepool {
    				
    				SetupStreamsPart2();
    			}});
    		}});
    	}
    	else if (delegateQueue && url != nil && [theDelegate respondsToSelector:@selector(socket:didConnectToUrl:)])
    	{
    		SetupStreamsPart1();
    		
    		dispatch_async(delegateQueue, ^{ @autoreleasepool {
    			
    			[theDelegate socket:self didConnectToUrl:url];
    			
    			dispatch_async(socketQueue, ^{ @autoreleasepool {
    				
    				SetupStreamsPart2();
    			}});
    		}});
    	}
    	else
    	{
    		SetupStreamsPart1();
    		SetupStreamsPart2();
    	}
    		
    	// Get the connected socket
    	
    	int socketFD = (socket4FD != SOCKET_NULL) ? socket4FD : (socket6FD != SOCKET_NULL) ? socket6FD : socketUN;
    	
    	// Enable non-blocking IO on the socket
    	
    	int result = fcntl(socketFD, F_SETFL, O_NONBLOCK);
    	if (result == -1)
    	{
    		NSString *errMsg = @"Error enabling non-blocking IO on socket (fcntl)";
    		[self closeWithError:[self otherError:errMsg]];
    		
    		return;
    	}
    	
    	// Setup our read/write sources
    	
    	[self setupReadAndWriteSourcesForNewlyConnectedSocket:socketFD];
    	
    	// Dequeue any pending read/write requests
    	
    	[self maybeDequeueRead];
    	[self maybeDequeueWrite];
    }
    
    - (void)didNotConnect:(int)aStateIndex error:(NSError *)error
    {
    	LogTrace();
    	
    	NSAssert(dispatch_get_specific(IsOnSocketQueueOrTargetQueueKey), @"Must be dispatched on socketQueue");
    	
    	
    	if (aStateIndex != stateIndex)
    	{
    		LogInfo(@"Ignoring didNotConnect, already disconnected");
    		
    		// The connect operation has been cancelled.
    		// That is, socket was disconnected, or connection has already timed out.
    		return;
    	}
    	
    	[self closeWithError:error];
    }


