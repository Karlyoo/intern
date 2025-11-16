## Practice of Communication Network_hw4 
- B11102012 
- B11102209 
### source code
---
#### server
```
#!/usr/bin/env python3
import socket
import threading
import sys

def handle_client(conn, addr, server_id):
    """
    Handles a single client connection, echoing back messages
    with the server's ID appended.
    """
    print(f"[SERVER {server_id}] Client connected from {addr}")
    try:
        while True:
            # Receive data from the client
            data = conn.recv(1024)
            if not data:
                break # Client disconnected

            message = data.decode('utf-8')
            print(f"[SERVER {server_id}] Received: {message}")

            # Create the response
            response = f"{message}server-{server_id}"
            
            # Send the response back
            conn.sendall(response.encode('utf-8'))
            
    except ConnectionResetError:
        print(f"[SERVER {server_id}] Client {addr} disconnected abruptly.")
    except Exception as e:
        print(f"[SERVER {server_id}] Error: {e}")
    finally:
        print(f"[SERVER {server_id}] Client {addr} disconnected.")
        conn.close()

def start_server(host, port, server_id):
    """
    Starts the echo server on a given host and port.
    """
    server_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    # Allow reusing the address to avoid "Address already in use" errors
    server_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    
    try:
        server_socket.bind((host, port))
        server_socket.listen(5)
        print(f"[SERVER {server_id}] Listening on {host}:{port}")

        while True:
            # Wait for a new client to connect
            client_socket, client_address = server_socket.accept()
            
            # Create a new thread to handle this client
            # This allows the server to handle multiple clients at once
            client_thread = threading.Thread(
                target=handle_client,
                args=(client_socket, client_address, server_id)
            )
            client_thread.daemon = True # Allows program to exit even if threads are running
            client_thread.start()

    except Exception as e:
        print(f"[SERVER {server_id}] Error: {e}")
    finally:
        server_socket.close()
        print(f"[SERVER {server_id}] Server shutting down.")

if __name__ == "__main__":
    if len(sys.argv) != 3:
        print("Usage: python3 server.py <server_id> <port>")
        sys.exit(1)
        
    server_id = sys.argv[1]
    port = int(sys.argv[2])
    
    start_server('0.0.0.0', port, server_id)
```
#### proxy
```
#!/usr/bin/env python3
import socket
import threading

# --- Configuration ---
PROXY_HOST = '0.0.0.0'
PROXY_PORT = 9090

# List of target servers the proxy will balance between
SERVERS = [
    ('localhost', 8001), # Server 1
    ('localhost', 8002)  # Server 2
]
# ---------------------

def handle_client(client_socket, client_addr):
    """
    Handles a single client connection, proxying its
    requests to servers in a round-robin fashion.
    """
    print(f"[PROXY] New client connected from {client_addr}")
    
    #
    # THIS IS THE KEY:
    # The server_index is a local variable inside this function.
    # Each client connection gets its own 'handle_client' thread,
    # so each client gets its own 'server_index' pointer.
    #
    server_index = 0

    try:
        while True:
            # 1. Receive a request from the client
            request_data = client_socket.recv(1024)
            if not request_data:
                break # Client disconnected

            # 2. Pick which server to send it to (Round-Robin)
            target_host, target_port = SERVERS[server_index]
            
            print(f"[PROXY] Client {client_addr} -> Server {server_index + 1} ({target_host}:{target_port})")

            response_data = b""
            try:
                # 3. Connect to the chosen server
                with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as server_socket:
                    server_socket.connect((target_host, target_port))
                    
                    # 4. Send the client's request to the server
                    server_socket.sendall(request_data)
                    
                    # 5. Receive the server's response
                    response_data = server_socket.recv(1024)
                    
            except Exception as e:
                print(f"[PROXY] Error connecting to server {target_host}:{target_port}: {e}")
                response_data = b"Error: Could not connect to backend server."

            # 6. Send the server's response back to the client
            client_socket.sendall(response_data)
            
            # 7. Update the round-robin pointer FOR THIS CLIENT
            server_index = (server_index + 1) % len(SERVERS)

    except ConnectionResetError:
        print(f"[PROXY] Client {client_addr} disconnected abruptly.")
    except Exception as e:
        print(f"[PROXY] Error: {e}")
    finally:
        print(f"[PROXY] Client {client_addr} disconnected.")
        client_socket.close()

def start_proxy():
    """
    Starts the main proxy server.
    """
    proxy_socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
    proxy_socket.setsockopt(socket.SOL_SOCKET, socket.SO_REUSEADDR, 1)
    
    try:
        proxy_socket.bind((PROXY_HOST, PROXY_PORT))
        proxy_socket.listen(5)
        print(f"[PROXY] TCP Proxy listening on {PROXY_HOST}:{PROXY_PORT}")
        print(f"[PROXY] Balancing between: {SERVERS}")

        while True:
            # Wait for a new client to connect
            client_socket, client_address = proxy_socket.accept()
            
            # Create a new thread for this client
            client_thread = threading.Thread(
                target=handle_client,
                args=(client_socket, client_address)
            )
            client_thread.daemon = True
            client_thread.start()

    except Exception as e:
        print(f"[PROXY] Error starting proxy: {e}")
    finally:
        proxy_socket.close()

if __name__ == "__main__":
    start_proxy()
```
#### client
```
#!/usr/bin/env python3
import socket
import time

PROXY_HOST = 'localhost'
PROXY_PORT = 9090
MESSAGES_TO_SEND = 4

def run_client():
    # Create a socket and maintain one connection for all messages
    try:
        with socket.socket(socket.AF_INET, socket.SOCK_STREAM) as s:
            
            # Connect to the PROXY, not the servers
            s.connect((PROXY_HOST, PROXY_PORT))
            print(f"Connected to proxy at {PROXY_HOST}:{PROXY_PORT}")

            for i in range(MESSAGES_TO_SEND):
                message = "hello"
                print(f"\nSending: '{message}' (Request {i + 1})")
                
                # Send the message
                s.sendall(message.encode('utf-8'))
                
                # Receive the response
                data = s.recv(1024)
                response = data.decode('utf-8')
                
                print(f"Received: '{response}'")
                
                # Wait 1 second before sending the next message
                time.sleep(1)
                
    except Exception as e:
        print(f"Error: {e}")
    finally:
        print("Client shutting down.")

if __name__ == "__main__":
    run_client()
```

### Simple instructions to run
#### server(2 terminal)
```
# server 1
python3 server.py 1 8001
```
```
# server 2
python3 server.py 2 8002
```
#### proxy
```
python3 proxy.py
```
#### client(2 terminal)
``` both
python3 client.py
```

### output
```
# log of server 1
karl@karl-ASUS-Vivobook-S-14-M3407HA-M3407HA:~$ python3 server.py 1 8001
[SERVER 1] Listening on 0.0.0.0:8001
[SERVER 1] Client connected from ('127.0.0.1', 52706)
[SERVER 1] Received: hello
[SERVER 1] Client ('127.0.0.1', 52706) disconnected.
[SERVER 1] Client connected from ('127.0.0.1', 52712)
[SERVER 1] Received: hello
[SERVER 1] Client ('127.0.0.1', 52712) disconnected.
[SERVER 1] Client connected from ('127.0.0.1', 52728)
[SERVER 1] Received: hello
[SERVER 1] Client ('127.0.0.1', 52728) disconnected.
[SERVER 1] Client connected from ('127.0.0.1', 52742)
[SERVER 1] Received: hello
[SERVER 1] Client ('127.0.0.1', 52742) disconnected.
```
```
# log of server 2
karl@karl-ASUS-Vivobook-S-14-M3407HA-M3407HA:~$ python3 server.py 2 8002
[SERVER 2] Listening on 0.0.0.0:8002
[SERVER 2] Client connected from ('127.0.0.1', 56604)
[SERVER 2] Received: hello
[SERVER 2] Client ('127.0.0.1', 56604) disconnected.
[SERVER 2] Client connected from ('127.0.0.1', 56608)
[SERVER 2] Received: hello
[SERVER 2] Client ('127.0.0.1', 56608) disconnected.
[SERVER 2] Client connected from ('127.0.0.1', 56616)
[SERVER 2] Received: hello
[SERVER 2] Client ('127.0.0.1', 56616) disconnected.
[SERVER 2] Client connected from ('127.0.0.1', 56618)
[SERVER 2] Received: hello
[SERVER 2] Client ('127.0.0.1', 56618) disconnected.
```

```
# log of proxy
karl@karl-ASUS-Vivobook-S-14-M3407HA-M3407HA:~$ python3 proxy.py
[PROXY] TCP Proxy listening on 0.0.0.0:9090
[PROXY] Balancing between: [('localhost', 8001), ('localhost', 8002)]
[PROXY] New client connected from ('127.0.0.1', 45124)
[PROXY] Client ('127.0.0.1', 45124) -> Server 1 (localhost:8001)
[PROXY] Client ('127.0.0.1', 45124) -> Server 2 (localhost:8002)
[PROXY] New client connected from ('127.0.0.1', 45134)
[PROXY] Client ('127.0.0.1', 45134) -> Server 1 (localhost:8001)
[PROXY] Client ('127.0.0.1', 45124) -> Server 1 (localhost:8001)
[PROXY] Client ('127.0.0.1', 45134) -> Server 2 (localhost:8002)
[PROXY] Client ('127.0.0.1', 45124) -> Server 2 (localhost:8002)
[PROXY] Client ('127.0.0.1', 45134) -> Server 1 (localhost:8001)
[PROXY] Client ('127.0.0.1', 45124) disconnected.
[PROXY] Client ('127.0.0.1', 45134) -> Server 2 (localhost:8002)
[PROXY] Client ('127.0.0.1', 45134) disconnected.
```

```
#log of client
karl@karl-ASUS-Vivobook-S-14-M3407HA-M3407HA:~$ python3 client.py
Connected to proxy at localhost:9090

Sending: 'hello' (Request 1)
Received: 'helloserver-1'

Sending: 'hello' (Request 2)
Received: 'helloserver-2'

Sending: 'hello' (Request 3)
Received: 'helloserver-1'

Sending: 'hello' (Request 4)
Received: 'helloserver-2'
Client shutting down.
```
