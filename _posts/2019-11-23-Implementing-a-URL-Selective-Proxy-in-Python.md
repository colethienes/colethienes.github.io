---
layout: post
title: Implementing a URL-Selective Proxy in Python
---

<p align="center">
<img src="https://github.com/colethienes/colethienes.github.io/raw/master/images/proxy.svg?sanitize=true" width="100%">
</p>

This article is somewhat of a continuation of my [previous article](https://colethienes.github.io/Managing-Machine-Learning-Dependencies-In-Distributed-Systems/) about the hurdles I faced while building [NBoost](https://github.com/koursaros-ai/nboost). A key feature of NBoost is that boosting search api results is **super easy**. In fact, all you need to do to is `pip install nboost` and make requests from the new proxy ip address instead of the Elasticsearch server. To do this, the proxy needs to capture certain urls (i.e. the `/_search` path for Elasticsearch) in order to alter (magnify) the client's request to the server and alter (rank) the upstream server's response. However, implementing this feature was deceptively complicated. Here are some of the issues I faced, and how I solved them:

### Detecting EOF

Logically, for the proxy to field requests and return responses, it needs to know when the client is done sending the request and when the server is done sending the response. The first approach I tried was using [asyncio](https://docs.python.org/3/library/asyncio-stream.html), which has an `at_eof()` method baked into it's [StreamReader](https://docs.python.org/3/library/asyncio-stream.html#streamreader). However, the protocol hung up the proxy in many of my test cases, so I scrapped it.

I realized the problem is that the standardization for detecting EOF in an http message is tricky. There are [three main methods](https://stackoverflow.com/questions/4824451/detect-end-of-http-request-body): 1. detect an empty buffer in the case of a `Transfer-Encoding: Chunked` header. 2. Use the `Content-Length` header. or 3. detect connection close (which only works for responses). 

To combat these difficult standards, I used the [Http C-Parser for NodeJS](https://github.com/nodejs/http-parser) (Specifically, I used [MagicStack's implementation](https://github.com/MagicStack/httptools)). This Http parser not only tells me when http messages are finished, but also has the added benefit of parsing the http message for me (and sending callbacks), so I can create search-api-specific parsers. The EOF detector looks like this:

```python
while not parser.is_done:
    data = in_socket.recv(self.bufsize)
    parser.feed(data)
``` 


### Capturing **Specific** URLs

What we (Koursaros) didn't want to have happen in production uses of NBoost was introduce complexity to their search api usage. If we're boosting search results, we didn't want the client to have to think about only sending search requests to the proxy, and sending the rest of the requests to the actual search engine server. They should be able to field **ALL** requests to the proxy.

Ideally, you would be able to check the socket message at the proxy, and if it's not a search request url (i.e. `/_search`), forward that message to the server socket. However, once you read from the socket, it's too late! What I ended up implementing was a **buffer**. All incoming messages to the proxy are buffered in memory. If it's a search request, the message is processed by the model, but if it's anything else, the entire buffer is dumped to the server and effectively proxied through.


### Show me the code!

This is the main proxy code for NBoost:

```python
class Proxy:
    def loop(self, client_socket, address):
        """Main ioloop for reranking server results to the client. Exceptions
        raised in the http parser must be reraised from __context__ because
        they are caught by the MagicStack implementation"""
        protocol = self.protocol()
        server_socket = socket.socket()
        buffer = dict(data=bytes())
        exception = None
    
        try:
            self.server_connect(server_socket)
            self.client_recv(client_socket, RequestHandler(protocol), buffer)
            self.server_send(server_socket, protocol.request)
            self.server_recv(server_socket, ResponseHandler(protocol))
            ranks = self.model_rank(protocol.topk, protocol.query, protocol.choices)
            protocol.on_rank(ranks)
    
        except HttpParserError as exc:
            exception = exc.__context__
    
        except Exception as exc:
            exception = exc
    
        try:
            log = '{}:{}: {}'.format(*address, protocol.request)
            if exception is None:
                self.logger.debug(log)
            else:
                self.logger.warning('%s: %s', type(exception).__name__, log)
                raise exception
    
        except StatusRequest:
            protocol.response.body = json.dumps(self.status, indent=2).encode()
            protocol.response.encode()
    
        except (UnknownRequest, MissingQuery):
            self.proxy_send(client_socket, server_socket, buffer)
            self.proxy_recv(client_socket, server_socket)
    
        except ResponseException:
            # allow the body to be sent back to the client
            pass
    
        except UpstreamConnectionError as exc:
            self.logger.error("Couldn't connect to server %s:%s...", *exc.args)
            protocol.on_error(exc)
    
        except Exception as exc:
            self.logger.error(repr(exc), exc_info=True)
            protocol.on_error(exc)
    
        finally:
            self.client_send(client_socket, protocol.response)
            client_socket.close()
            server_socket.close()
    
    def worker(self):
        """Socket loop for each worker"""
        try:
            while True:
                self.loop(*self.sock.accept())
    
        except OSError:
            self.logger.debug('Closing worker %s...', get_ident())
```

Essentially, each worker (thread) executes `worker()` and enters an infinite loop of accepting client requests. Every time a worker gets a client request from the `accept()` method, it initializes a buffer to store the message. The `client_recv` function reads incoming data from the client to the buffer and also to the parser. When the parser gets to the url, the proxy checks whether it's a search request. If it's not, an `UnknownRequest` exception is raised and the original client request is just passed through to the server.

If you have any questions, comments, or think I missed anything, feel free to reach out!
 