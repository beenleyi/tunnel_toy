# Python程序设计#4作业

截止时间：2020年11月16日23:59:59

## 作业题目

在作业#3的基础上实现localProxy命令行参数账号登录

在作业#3的基础上实现remoteProxy多账号认证

remoteProxy采用SQLite3数据库进行用户账号管理（用户名、密码）

remoteProxy使用aiosqlite操作SQLite3数据库

## 作业内容

程序源代码嵌入下方的code block中。

local_server.py

```python
from enum import Enum
import asyncio
import struct
import socket
import hashlib
import signal
import logging
import argparse
import sys
import traceback
import ipaddress
SOCKS_VER = 5

READ_MODE = Enum(
    'readmode', ('EXACT', 'LINE', 'MAX', 'UNTIL')
)

class program_err(Exception):
    print(Exception)


def logExc(exc):
    if args.logExc:
        logger.error(f'{traceback.format_exc()}')


async def handle_local(client_reader, remote_writer, client_writer):
    while True:
        try:
            req_data = await aio_read(client_reader, READ_MODE.MAX, read_len=4096)
            if not req_data:
                return
            client_addr = client_writer.get_extra_info('peername')
            logger.debug('client {} want: {}'.format(client_addr, req_data[0:8]))
            await aio_write(remote_writer,req_data)
        except Exception as exc:
            logger.debug(exc)


async def handle_remote(client_writer, remote_reader, remote_writer):
    while True:
        try:
            resp_data = await aio_read(remote_reader, READ_MODE.MAX, read_len=4096)
            if not resp_data:
                return
            server_addr = remote_writer.get_extra_info('peername')
            logger.debug('server {} resp: {}'.format(server_addr, resp_data[0:8]))
            await aio_write(client_writer,resp_data)
        except Exception as exc:
            logger.debug(exc)


async def aio_read(reader, read_mode, *, log_hint=None, exact_data=None, read_len=None, until_str=b'\r\n'):
    data = None
    try:
        if read_mode == READ_MODE.EXACT:
            if exact_data:
                read_len = len(exact_data)
            data = await reader.readexactly(read_len)
            if exact_data and data != exact_data:
                raise program_err(
                    f'ERR={data}, it should be {exact_data}, {log_hint}')
        elif read_mode == READ_MODE.LINE:
            data = await reader.readline()
        elif read_mode == READ_MODE.MAX:
            data = await reader.read(read_len)
        elif read_mode == READ_MODE.UNTIL:
            data = await reader.readuntil(until_str)
        else:
            logger.error(f'invalid mode={read_mode}')
            exit(1)
    except asyncio.IncompleteReadError as exc:
        raise program_err(f'EXC={exc} {log_hint}')
    except ConnectionResetError as exc:
        raise program_err(f'EXC={exc} {log_hint}')
    except ConnectionAbortedError as exc:
        raise program_err(f'EXC={exc} {log_hint}')
    if not data:
        raise program_err(f'find EOF when read {log_hint}')
    return data

async def aio_write(writer, data=None, *, log_hint=None):
    try:
        writer.write(data)
        await writer.drain()
    except ConnectionAbortedError as exc:
        raise program_errk(f'EXC={exc} {log_hint}')
    except Exception as exc:
        logger.debug(exc)


async def handle(client_reader, client_writer):
    client_host, client_port, *_ = client_writer.get_extra_info('peername')
    logger.info(f'Request from local: {client_host} {client_port}')
    first_byte = await aio_read(client_reader, READ_MODE.EXACT, read_len=1, log_hint=f'first byte from {client_host} {client_port}')
    log_hint=f'{client_host} {client_port}'
    remote_host = None
    remote_port=None
    try:
        if first_byte == b'\x05':
            proxy_protocal = 'SOCKS5'
            nmethods = await aio_read(client_reader, READ_MODE.EXACT, read_len=1, log_hint=f'nmethods')
            await aio_read(client_reader, READ_MODE.EXACT, read_len=nmethods[0], log_hint='methods')
            resp_data = struct.pack("!BB", SOCKS_VER, 0)
            await aio_write(client_writer, b'\x05\x00', log_hint='reply no auth')
            await aio_read(client_reader, READ_MODE.EXACT, exact_data=b'\x05\x01\x00', read_len=1, log_hint='version command reservation')
            atyp = await aio_read(client_reader, READ_MODE.EXACT, read_len=1, log_hint=f'atyp')
            if atyp == b'\x01':  # IPv4
                temp_addr = await aio_read(client_reader, READ_MODE.EXACT, read_len=4, log_hint='ipv4')
                remote_host = str(ipaddress.ip_address(dstHost))
            elif atyp == b'\x03':  # domain
                domain_len = await aio_read(client_reader, READ_MODE.EXACT, read_len=1, log_hint='domain len')
                remote_host = await aio_read(client_reader, READ_MODE.EXACT, read_len=domain_len[0], log_hint='domain')
                remote_host = remote_host.decode('utf8')
            elif atyp == b'\x04':  # IPv6
                temp_addr = await aio_read(client_reader, READ_MODE.EXACT, read_len=16, log_hint='ipv6')
                remote_host = str(ipaddress.ip_address(dstHost))
            else:
                raise program_err(f'invalid atyp')
            remote_port = await aio_read(client_reader, READ_MODE.EXACT, read_len=2, log_hint='port')
            remote_port = int.from_bytes(remote_port, 'big')            
        else:
            req = await aio_read(client_reader, READ_MODE.LINE, log_hint='http request')
            req = bytes.decode(first_byte+req)
            method, uri, protocal, *_ = req.split()
            if method.lower() == 'connect':
                proxy_protocal = 'HTTPS'
                log_hint=f'{log_hint} {proxy_protocal}'
                remote_host, remote_port, *_ = uri.split(':')
                await aio_read(client_reader, READ_MODE.UNTIL, until_str=b'\r\n\r\n', log_hint='message left')
            else:
                raise program_err(f'cannot server the request {req.split()}')
        
        logger.info(f'{log_hint} connect to {remote_host} {remote_port}')

        remote_reader, remote_writer = await asyncio.open_connection(args.remote_server_ip, args.remote_server_port)
        await aio_write(remote_writer, f'{remote_host} {remote_port} {args.username} {password}\r\n'.encode(),
            log_hint=f'{log_hint} connect to remote server, {remote_host} {remote_port}')
        reply_bindaddr = await aio_read(remote_reader, READ_MODE.LINE, log_hint='remote server reply the bind addr')
        reply_bindaddr = bytes.decode(reply_bindaddr)
        addr = reply_bindaddr[:-2].split()
        bind_host, bind_port=addr[0],addr[1]
        logger.info(f'{log_hint} bind at {bind_host} {bind_port}')
        if proxy_protocal == 'SOCKS5':
            bind_domain=bind_host
            bind_host = ipaddress.ip_address(bind_host)
            atyp = b'\x03'
            host_data=None
            try:
                if bind_host.version == 4:
                    atyp = b'\x01'
                    host_data = struct.pack('!L', int(bind_host))
                else:
                    atyp = b'\x04'
                    host_data = struct.pack('!16s', ipaddress.v6_int_to_packed(int(bind_host)))
            except Exception as exc:
                host_data = struct.pack(f'!B{len(bind_domain)}s', len(bind_domain), bind_domain.encode())
            reply_data = struct.pack(f'!ssss{len(host_data)}sH', b'\x05', b'\x00', b'\x00', atyp, host_data, bind_port)
            await aio_write(client_writer, reply_data, log_hint='reply the bind addr')  
        else:
            await aio_write(client_writer, f'{protocal} 200 OK\r\n\r\n'.encode(), log_hint='response to HTTPS')
            
        try:
            await asyncio.gather(handle_local(client_reader, remote_writer, client_writer), handle_remote(client_writer, remote_reader, remote_writer))
        except Exception as exc:
                logExc(exc)
                client_writer.close()
                remote_writer.close()
    except program_err as exc:
        logger.info(f'{log_hint} {exc}')
        await client_writer.close()
        await remote_writer.close()
    except OSError:
        log.info(f'{log_hint} connect fail')
        await client_writer.close()
    except Exception as exc:
        logger.error(f'{traceback.format_exc()}')
        exit(1)

async def main():
    server = await asyncio.start_server(
        handle, host=args.listen_ip, port=args.listen_port)
    addr = server.sockets[0].getsockname()
    logger.info(f'Serving on {addr[1]}')
    async with server:
        await server.serve_forever()


if __name__ == '__main__':

    # interrupt from keyboard, perform the default function for the signal
    signal.signal(signal.SIGINT, signal.SIG_DFL)

    # logging
    logger = logging.getLogger(__name__)
    logger.setLevel(level=logging.DEBUG)
    handler = logging.FileHandler('local_server.log')
    formatter = logging.Formatter(
        '%(asctime)s %(levelname).1s %(lineno)-3d %(funcName)-20s %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    chlr = logging.StreamHandler()
    logger.addHandler(chlr)

    # parser
    _parser = argparse.ArgumentParser(description='server')
    _parser.add_argument('--exc', dest='logExc', default=False,
                         action='store_true', help='show exception traceback')
    _parser.add_argument('--listen_ip', dest='listen_ip', metavar='listen_host',
                         help='proxy listen host default listen all interfaces')
    _parser.add_argument('--listen_port', dest='listen_port',
                         metavar='listen_port', required=True, help='proxy listen port')
    _parser.add_argument('--remote_ip', dest='remote_server_ip',
                         metavar='remote_server_ip', required=True, help='remote server ip')
    _parser.add_argument('--remote_port', dest='remote_server_port',
                         metavar='remote_server_port', required=True, help='remote server port')
    _parser.add_argument('--user', dest='username',
                         metavar='username', required=True, help='username')
    args = _parser.parse_args()
    logger.debug(f'{args}')

    password = input("please input your password:\n")

    if sys.platform == 'win32':
        asyncio.set_event_loop(asyncio.ProactorEventLoop())

    asyncio.run(main())
```

remote_server.py

```python
from enum import Enum
import asyncio
import struct
import aiosqlite3
import socket
import hashlib
import signal
import logging
import argparse
import sys
import traceback
SOCKS_VER = 5

READ_MODE = Enum(
    'readmode', ('EXACT', 'LINE', 'MAX', 'UNTIL')
)

class program_err(Exception):
    print(Exception)


def logExc(exc):
    if args.logExc:
        logger.error(f'{traceback.format_exc()}')


async def handle_local(client_reader, remote_writer, client_writer):
    while True:
        try:
            req_data = await aio_read(client_reader, READ_MODE.MAX, read_len=4096)
            if not req_data:
                return
            client_addr = client_writer.get_extra_info('peername')
            logger.debug('client {} want: {}'.format(client_addr, req_data[0:8]))
            await aio_write(remote_writer,req_data)
        except Exception as exc:
            logger.debug(exc)


async def handle_remote(client_writer, remote_reader, remote_writer):
    while True:
        try:
            resp_data = await aio_read(remote_reader, READ_MODE.MAX, read_len=4096)
            if not resp_data:
                return
            server_addr = remote_writer.get_extra_info('peername')
            logger.debug('server {} resp: {}'.format(server_addr, resp_data[0:8]))
            await aio_write(client_writer,resp_data)
        except Exception as exc:
            logger.debug(exc)


async def aio_read(reader, read_mode, *, log_hint=None, exact_data=None, read_len=None, until_str=b'\r\n'):
    data = None
    try:
        if read_mode == READ_MODE.EXACT:
            if exact_data:
                read_len = len(exact_data)
            data = await reader.readexactly(read_len)
            if exact_data and data != exact_data:
                raise program_err(
                    f'ERR={data}, it should be {exact_data}, {log_hint}')
        elif read_mode == READ_MODE.LINE:
            data = await reader.readline()
        elif read_mode == READ_MODE.MAX:
            data = await reader.read(read_len)
        elif read_mode == READ_MODE.UNTIL:
            data = await reader.readuntil(until_str)
        else:
            logger.error(f'invalid mode={read_mode}')
            exit(1)
    except asyncio.IncompleteReadError as exc:
        raise program_err(f'EXC={exc} {log_hint}')
    except ConnectionResetError as exc:
        raise program_err(f'EXC={exc} {log_hint}')
    except ConnectionAbortedError as exc:
        raise program_err(f'EXC={exc} {log_hint}')
    if not data:
        raise program_err(f'find EOF when read {log_hint}')
    return data

async def aio_write(writer, data=None, *, log_hint=None):
    try:
        writer.write(data)
        await writer.drain()
    except ConnectionAbortedError as exc:
        raise program_errk(f'EXC={exc} {log_hint}')
    except Exception as exc:
        logger.debug(exc)

async def handle(client_reader, client_writer):
    req = await aio_read(client_reader,READ_MODE.LINE,log_hint='recv req')
    req = bytes.decode(req)
    addr = req[:-2].split()
    logger.info(f'remote server receive request {req[:-2]}')
    host, port = addr[0], int(addr[1])
    username, password = addr[2], addr[3]
    cursor = await db.execute("select password from user where username='%s'" % username)
    row = await cursor.fetchone()
    assert password==row[0]
    remote_reader, remote_writer = await asyncio.open_connection(host, port)
    bind_host, bind_port, *_ = remote_writer.get_extra_info('sockname')
    logger.debug(f'connect to {host} {port}, with {bind_host} {bind_port}')
    await aio_write(client_writer,f'{bind_host} {bind_port}\r\n'.encode())
    try:
        await asyncio.gather(handle_local(client_reader, remote_writer,client_writer), handle_remote(client_writer, remote_reader,remote_writer))
    except Exception as exc:
        logExc(exc)
        client_writer.close()
        remote_writer.close()

async def main():
    global db
    db=await aiosqlite3.connect("user.db")
    server = await asyncio.start_server(
        handle, host=args.listen_host, port=args.listen_port)
    addr = server.sockets[0].getsockname()
    logger.info(f'Serving on {addr}')
    async with server:
        await server.serve_forever()

if __name__ == '__main__':
    # interrupt from keyboard, perform the default function for the signal
    signal.signal(signal.SIGINT, signal.SIG_DFL)
    
    # logging
    logger = logging.getLogger(__name__)
    logger.setLevel(level=logging.DEBUG)
    handler = logging.FileHandler('remote_server.log')
    formatter = logging.Formatter('%(asctime)s %(levelname).1s %(lineno)-3d %(funcName)-20s %(message)s')
    handler.setFormatter(formatter)
    logger.addHandler(handler)
    chlr = logging.StreamHandler()
    logger.addHandler(chlr)
    # parser
    _parser = argparse.ArgumentParser(description='server')
    _parser.add_argument('--exc', dest='logExc', default=False, action='store_true', help='show exception traceback')
    _parser.add_argument('--listen_ip', dest='listen_host', metavar='listen_host', help='proxy listen host default listen all interfaces')
    _parser.add_argument('--listen_port', dest='listen_port', metavar='listen_port', required=True, help='proxy listen port')
    args = _parser.parse_args()

    if sys.platform == 'win32':
        asyncio.set_event_loop(asyncio.ProactorEventLoop())

    asyncio.run(main())
```



## 代码说明（可选）

源代码中不要出现大段的说明注释，如果需要可以可以在本节中加上说明。

**使用数据库验证用户名密码的实现：**

- remote_server.py中114-115行建立数据库连接；
- remote_server.py中99-101行验证密码是否正确；
- local_server.py中217-202行请求输入用户名和密码，其中用户名是运行参数之一
- local_server.py中137行对远程服务器发送域名和端口的同时发送用户名和密码

**本周对代码进行了较为完整的改善，有如下修改：**

- 完成读写函数，代码看起来整洁不少
- 跟远程服务器沟通时，增加远程服务器回复其代理的地址和端口，这更符合SOCKS5回复的要求
- 完善exception报错
- 在SOCKS5和HTTPS可以共用代码的地方合并

