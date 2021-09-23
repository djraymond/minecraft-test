# minecraft-test
minecraft test


Permission is hereby granted, free of charge, to any person obtaining a copy
of this software and associated documentation files (the "Software"), to deal
in the Software without restriction, including without limitation the rights
to use, copy, modify, merge, publish, distribute, sublicense, and/or sell
copies of the Software, and to permit persons to whom the Software is
furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all
copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED "AS IS", WITHOUT WARRANTY OF ANY KIND, EXPRESS OR
IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY,
FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE
AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER
LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM,
OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE
SOFTWARE.

## Craft

Minecraft clone for Windows, Mac OS X and Linux. Just a few thousand lines of C using modern OpenGL (shaders). Online multiplayer support is included using a Python-based server.

http://www.michaelfogleman.com/craft/

![Screenshot](https://i.imgur.com/SH7wcas.png)

### Features

* Simple but nice looking terrain generation using perlin / simplex noise.
* More than 10 types of blocks and more can be added easily.
* Supports plants (grass, flowers, trees, etc.) and transparency (glass).
* Simple clouds in the sky (they don't move).
* Day / night cycles and a textured sky dome.
* World changes persisted in a sqlite3 database.
* Multiplayer support!

### Download

Mac and Windows binaries are available on the website.

http://www.michaelfogleman.com/craft/

See below to run from source.

### Install Dependencies

#### Mac OS X

Download and install [CMake](http://www.cmake.org/cmake/resources/software.html)
if you don't already have it. You may use [Homebrew](http://brew.sh) to simplify
the installation:

    brew install cmake

#### Linux (Ubuntu)

    sudo apt-get install cmake libglew-dev xorg-dev libcurl4-openssl-dev
    sudo apt-get build-dep glfw

#### Windows

Download and install [CMake](http://www.cmake.org/cmake/resources/software.html)
and [MinGW](http://www.mingw.org/). Add `C:\MinGW\bin` to your `PATH`.

Download and install [cURL](http://curl.haxx.se/download.html) so that
CURL/lib and CURL/include are in your Program Files directory.

Use the following commands in place of the ones described in the next section.

    cmake -G "MinGW Makefiles"
    mingw32-make

### Compile and Run

Once you have the dependencies (see above), run the following commands in your
terminal.

    git clone https://github.com/fogleman/Craft.git
    cd Craft
    cmake .
    make
    ./craft

### Multiplayer

After many years, craft.michaelfogleman.com has been taken down. See the [Server](#server) section for info on self-hosting.

#### Client

You can connect to a server with command line arguments...

```bash
./craft [HOST [PORT]]
```

Or, with the "/online" command in the game itself.
    
    /online [HOST [PORT]]

#### Server

You can run your own server or connect to mine. The server is written in Python
but requires a compiled DLL so it can perform the terrain generation just like
the client.

```bash
gcc -std=c99 -O3 -fPIC -shared -o world -I src -I deps/noise deps/noise/noise.c src/world.c
python server.py [HOST [PORT]]
```

### Controls

- WASD to move forward, left, backward, right.
- Space to jump.
- Left Click to destroy a block.
- Right Click or Cmd + Left Click to create a block.
- Ctrl + Right Click to toggle a block as a light source.
- 1-9 to select the block type to create.
- E to cycle through the block types.
- Tab to toggle between walking and flying.
- ZXCVBN to move in exact directions along the XYZ axes.
- Left shift to zoom.
- F to show the scene in orthographic mode.
- O to observe players in the main view.
- P to observe players in the picture-in-picture view.
- T to type text into chat.
- Forward slash (/) to enter a command.
- Backquote (`) to write text on any block (signs).
- Arrow keys emulate mouse movement.
- Enter emulates mouse click.

### Chat Commands

    /goto [NAME]

Teleport to another user.
If NAME is unspecified, a random user is chosen.

    /list

Display a list of connected users.

    /login NAME

Switch to another registered username.
The login server will be re-contacted. The username is case-sensitive.

    /logout

Unauthenticate and become a guest user.
Automatic logins will not occur again until the /login command is re-issued.

    /offline [FILE]

Switch to offline mode.
FILE specifies the save file to use and defaults to "craft".

    /online HOST [PORT]

Connect to the specified server.

    /pq P Q

Teleport to the specified chunk.

    /spawn

Teleport back to the spawn point.

### Screenshot

![Screenshot](https://i.imgur.com/foYz3aN.png)

### Implementation Details

#### Terrain Generation

The terrain is generated using Simplex noise - a deterministic noise function seeded based on position. So the world will always be generated the same way in a given location.

The world is split up into 32x32 block chunks in the XZ plane (Y is up). This allows the world to be “infinite” (floating point precision is currently a problem at large X or Z values) and also makes it easier to manage the data. Only visible chunks need to be queried from the database.

#### Rendering

Only exposed faces are rendered. This is an important optimization as the vast majority of blocks are either completely hidden or are only exposing one or two faces. Each chunk records a one-block width overlap for each neighboring chunk so it knows which blocks along its perimeter are exposed.

Only visible chunks are rendered. A naive frustum-culling approach is used to test if a chunk is in the camera’s view. If it is not, it is not rendered. This results in a pretty decent performance improvement as well.

Chunk buffers are completely regenerated when a block is changed in that chunk, instead of trying to update the VBO.

Text is rendered using a bitmap atlas. Each character is rendered onto two triangles forming a 2D rectangle.

“Modern” OpenGL is used - no deprecated, fixed-function pipeline functions are used. Vertex buffer objects are used for position, normal and texture coordinates. Vertex and fragment shaders are used for rendering. Matrix manipulation functions are in matrix.c for translation, rotation, perspective, orthographic, etc. matrices. The 3D models are made up of very simple primitives - mostly cubes and rectangles. These models are generated in code in cube.c.

Transparency in glass blocks and plants (plants don’t take up the full rectangular shape of their triangle primitives) is implemented by discarding magenta-colored pixels in the fragment shader.

#### Database

User changes to the world are stored in a sqlite database. Only the delta is stored, so the default world is generated and then the user changes are applied on top when loading.

The main database table is named “block” and has columns p, q, x, y, z, w. (p, q) identifies the chunk, (x, y, z) identifies the block position and (w) identifies the block type. 0 represents an empty block (air).

In game, the chunks store their blocks in a hash map. An (x, y, z) key maps to a (w) value.

The y-position of blocks are limited to 0 <= y < 256. The upper limit is mainly an artificial limitation to prevent users from building unnecessarily tall structures. Users are not allowed to destroy blocks at y = 0 to avoid falling underneath the world.

#### Multiplayer

Multiplayer mode is implemented using plain-old sockets. A simple, ASCII, line-based protocol is used. Each line is made up of a command code and zero or more comma-separated arguments. The client requests chunks from the server with a simple command: C,p,q,key. “C” means “Chunk” and (p, q) identifies the chunk. The key is used for caching - the server will only send block updates that have been performed since the client last asked for that chunk. Block updates (in realtime or as part of a chunk request) are sent to the client in the format: B,p,q,x,y,z,w. After sending all of the blocks for a requested chunk, the server will send an updated cache key in the format: K,p,q,key. The client will store this key and use it the next time it needs to ask for that chunk. Player positions are sent in the format: P,pid,x,y,z,rx,ry. The pid is the player ID and the rx and ry values indicate the player’s rotation in two different axes. The client interpolates player positions from the past two position updates for smoother animation. The client sends its position to the server at most every 0.1 seconds (less if not moving).

Client-side caching to the sqlite database can be performance intensive when connecting to a server for the first time. For this reason, sqlite writes are performed on a background thread. All writes occur in a transaction for performance. The transaction is committed every 5 seconds as opposed to some logical amount of work completed. A ring / circular buffer is used as a queue for what data is to be written to the database.

In multiplayer mode, players can observe one another in the main view or in a picture-in-picture view. Implementation of the PnP was surprisingly simple - just change the viewport and render the scene again from the other player’s point of view.

#### Collision Testing

Hit testing (what block the user is pointing at) is implemented by scanning a ray from the player’s position outward, following their sight vector. This is not a precise method, so the step rate can be made smaller to be more accurate.

Collision testing simply adjusts the player’s position to remain a certain distance away from any adjacent blocks that are obstacles. (Clouds and plants are not marked as obstacles, so you pass right through them.)

#### Sky Dome

A textured sky dome is used for the sky. The X-coordinate of the texture represents time of day. The Y-values map from the bottom of the sky sphere to the top of the sky sphere. The player is always in the center of the sphere. The fragment shaders for the blocks also sample the sky texture to determine the appropriate fog color to blend with based on the block’s position relative to the backing sky.

#### Ambient Occlusion

Ambient occlusion is implemented as described on this page:

http://0fps.wordpress.com/2013/07/03/ambient-occlusion-for-minecraft-like-worlds/

#### Dependencies

* GLEW is used for managing OpenGL extensions across platforms.
* GLFW is used for cross-platform window management.
* CURL is used for HTTPS / SSL POST for the authentication process.
* lodepng is used for loading PNG textures.
* sqlite3 is used for saving the blocks added / removed by the user.
* tinycthread is used for cross-platform threading.

from math import floor
from world import World
import Queue
import SocketServer
import datetime
import random
import re
import requests
import sqlite3
import sys
import threading
import time
import traceback

DEFAULT_HOST = '0.0.0.0'
DEFAULT_PORT = 4080

DB_PATH = 'craft.db'
LOG_PATH = 'log.txt'

CHUNK_SIZE = 32
BUFFER_SIZE = 4096
COMMIT_INTERVAL = 5

AUTH_REQUIRED = True
AUTH_URL = 'https://craft.michaelfogleman.com/api/1/access'

DAY_LENGTH = 600
SPAWN_POINT = (0, 0, 0, 0, 0)
RATE_LIMIT = False
RECORD_HISTORY = False
INDESTRUCTIBLE_ITEMS = set([16])
ALLOWED_ITEMS = set([
    0, 1, 2, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15,
    17, 18, 19, 20, 21, 22, 23,
    32, 33, 34, 35, 36, 37, 38, 39, 40, 41, 42, 43, 44, 45, 46, 47,
    48, 49, 50, 51, 52, 53, 54, 55, 56, 57, 58, 59, 60, 61, 62, 63])

AUTHENTICATE = 'A'
BLOCK = 'B'
CHUNK = 'C'
DISCONNECT = 'D'
KEY = 'K'
LIGHT = 'L'
NICK = 'N'
POSITION = 'P'
REDRAW = 'R'
SIGN = 'S'
TALK = 'T'
TIME = 'E'
VERSION = 'V'
YOU = 'U'

try:
    from config import *
except ImportError:
    pass

def log(*args):
    now = datetime.datetime.utcnow()
    line = ' '.join(map(str, (now,) + args))
    print line
    with open(LOG_PATH, 'a') as fp:
        fp.write('%s\n' % line)

def chunked(x):
    return int(floor(round(x) / CHUNK_SIZE))

def packet(*args):
    return '%s\n' % ','.join(map(str, args))

class RateLimiter(object):
    def __init__(self, rate, per):
        self.rate = float(rate)
        self.per = float(per)
        self.allowance = self.rate
        self.last_check = time.time()
    def tick(self):
        if not RATE_LIMIT:
            return False
        now = time.time()
        elapsed = now - self.last_check
        self.last_check = now
        self.allowance += elapsed * (self.rate / self.per)
        if self.allowance > self.rate:
            self.allowance = self.rate
        if self.allowance < 1:
            return True # too fast
        else:
            self.allowance -= 1
            return False # okay

class Server(SocketServer.ThreadingMixIn, SocketServer.TCPServer):
    allow_reuse_address = True
    daemon_threads = True

class Handler(SocketServer.BaseRequestHandler):
    def setup(self):
        self.position_limiter = RateLimiter(100, 5)
        self.limiter = RateLimiter(1000, 10)
        self.version = None
        self.client_id = None
        self.user_id = None
        self.nick = None
        self.queue = Queue.Queue()
        self.running = True
        self.start()
    def handle(self):
        model = self.server.model
        model.enqueue(model.on_connect, self)
        try:
            buf = []
            while True:
                data = self.request.recv(BUFFER_SIZE)
                if not data:
                    break
                buf.extend(data.replace('\r\n', '\n'))
                while '\n' in buf:
                    index = buf.index('\n')
                    line = ''.join(buf[:index])
                    buf = buf[index + 1:]
                    if not line:
                        continue
                    if line[0] == POSITION:
                        if self.position_limiter.tick():
                            log('RATE', self.client_id)
                            self.stop()
                            return
                    else:
                        if self.limiter.tick():
                            log('RATE', self.client_id)
                            self.stop()
                            return
                    model.enqueue(model.on_data, self, line)
        finally:
            model.enqueue(model.on_disconnect, self)
    def finish(self):
        self.running = False
    def stop(self):
        self.request.close()
    def start(self):
        thread = threading.Thread(target=self.run)
        thread.setDaemon(True)
        thread.start()
    def run(self):
        while self.running:
            try:
                buf = []
                try:
                    buf.append(self.queue.get(timeout=5))
                    try:
                        while True:
                            buf.append(self.queue.get(False))
                    except Queue.Empty:
                        pass
                except Queue.Empty:
                    continue
                data = ''.join(buf)
                self.request.sendall(data)
            except Exception:
                self.request.close()
                raise
    def send_raw(self, data):
        if data:
            self.queue.put(data)
    def send(self, *args):
        self.send_raw(packet(*args))

class Model(object):
    def __init__(self, seed):
        self.world = World(seed)
        self.clients = []
        self.queue = Queue.Queue()
        self.commands = {
            AUTHENTICATE: self.on_authenticate,
            CHUNK: self.on_chunk,
            BLOCK: self.on_block,
            LIGHT: self.on_light,
            POSITION: self.on_position,
            TALK: self.on_talk,
            SIGN: self.on_sign,
            VERSION: self.on_version,
        }
        self.patterns = [
            (re.compile(r'^/nick(?:\s+([^,\s]+))?$'), self.on_nick),
            (re.compile(r'^/spawn$'), self.on_spawn),
            (re.compile(r'^/goto(?:\s+(\S+))?$'), self.on_goto),
            (re.compile(r'^/pq\s+(-?[0-9]+)\s*,?\s*(-?[0-9]+)$'), self.on_pq),
            (re.compile(r'^/help(?:\s+(\S+))?$'), self.on_help),
            (re.compile(r'^/list$'), self.on_list),
        ]
    def start(self):
        thread = threading.Thread(target=self.run)
        thread.setDaemon(True)
        thread.start()
    def run(self):
        self.connection = sqlite3.connect(DB_PATH)
        self.create_tables()
        self.commit()
        while True:
            try:
                if time.time() - self.last_commit > COMMIT_INTERVAL:
                    self.commit()
                self.dequeue()
            except Exception:
                traceback.print_exc()
    def enqueue(self, func, *args, **kwargs):
        self.queue.put((func, args, kwargs))
    def dequeue(self):
        try:
            func, args, kwargs = self.queue.get(timeout=5)
            func(*args, **kwargs)
        except Queue.Empty:
            pass
    def execute(self, *args, **kwargs):
        return self.connection.execute(*args, **kwargs)
    def commit(self):
        self.last_commit = time.time()
        self.connection.commit()
    def create_tables(self):
        queries = [
            'create table if not exists block ('
            '    p int not null,'
            '    q int not null,'
            '    x int not null,'
            '    y int not null,'
            '    z int not null,'
            '    w int not null'
            ');',
            'create unique index if not exists block_pqxyz_idx on '
            '    block (p, q, x, y, z);',
            'create table if not exists light ('
            '    p int not null,'
            '    q int not null,'
            '    x int not null,'
            '    y int not null,'
            '    z int not null,'
            '    w int not null'
            ');',
            'create unique index if not exists light_pqxyz_idx on '
            '    light (p, q, x, y, z);',
            'create table if not exists sign ('
            '    p int not null,'
            '    q int not null,'
            '    x int not null,'
            '    y int not null,'
            '    z int not null,'
            '    face int not null,'
            '    text text not null'
            ');',
            'create index if not exists sign_pq_idx on sign (p, q);',
            'create unique index if not exists sign_xyzface_idx on '
            '    sign (x, y, z, face);',
            'create table if not exists block_history ('
            '   timestamp real not null,'
            '   user_id int not null,'
            '   x int not null,'
            '   y int not null,'
            '   z int not null,'
            '   w int not null'
            ');',
        ]
        for query in queries:
            self.execute(query)
    def get_default_block(self, x, y, z):
        p, q = chunked(x), chunked(z)
        chunk = self.world.get_chunk(p, q)
        return chunk.get((x, y, z), 0)
    def get_block(self, x, y, z):
        query = (
            'select w from block where '
            'p = :p and q = :q and x = :x and y = :y and z = :z;'
        )
        p, q = chunked(x), chunked(z)
        rows = list(self.execute(query, dict(p=p, q=q, x=x, y=y, z=z)))
        if rows:
            return rows[0][0]
        return self.get_default_block(x, y, z)
    def next_client_id(self):
        result = 1
        client_ids = set(x.client_id for x in self.clients)
        while result in client_ids:
            result += 1
        return result
    def on_connect(self, client):
        client.client_id = self.next_client_id()
        client.nick = 'guest%d' % client.client_id
        log('CONN', client.client_id, *client.client_address)
        client.position = SPAWN_POINT
        self.clients.append(client)
        client.send(YOU, client.client_id, *client.position)
        client.send(TIME, time.time(), DAY_LENGTH)
        client.send(TALK, 'Welcome to Craft!')
        client.send(TALK, 'Type "/help" for a list of commands.')
        self.send_position(client)
        self.send_positions(client)
        self.send_nick(client)
        self.send_nicks(client)
    def on_data(self, client, data):
        #log('RECV', client.client_id, data)
        args = data.split(',')
        command, args = args[0], args[1:]
        if command in self.commands:
            func = self.commands[command]
            func(client, *args)
    def on_disconnect(self, client):
        log('DISC', client.client_id, *client.client_address)
        self.clients.remove(client)
        self.send_disconnect(client)
        self.send_talk('%s has disconnected from the server.' % client.nick)
    def on_version(self, client, version):
        if client.version is not None:
            return
        version = int(version)
        if version != 1:
            client.stop()
            return
        client.version = version
        # TODO: client.start() here
    def on_authenticate(self, client, username, access_token):
        user_id = None
        if username and access_token:
            payload = {
                'username': username,
                'access_token': access_token,
            }
            response = requests.post(AUTH_URL, data=payload)
            if response.status_code == 200 and response.text.isdigit():
                user_id = int(response.text)
        client.user_id = user_id
        if user_id is None:
            client.nick = 'guest%d' % client.client_id
            client.send(TALK, 'Visit craft.michaelfogleman.com to register!')
        else:
            client.nick = username
        self.send_nick(client)
        # TODO: has left message if was already authenticated
        self.send_talk('%s has joined the game.' % client.nick)
    def on_chunk(self, client, p, q, key=0):
        packets = []
        p, q, key = map(int, (p, q, key))
        query = (
            'select rowid, x, y, z, w from block where '
            'p = :p and q = :q and rowid > :key;'
        )
        rows = self.execute(query, dict(p=p, q=q, key=key))
        max_rowid = 0
        blocks = 0
        for rowid, x, y, z, w in rows:
            blocks += 1
            packets.append(packet(BLOCK, p, q, x, y, z, w))
            max_rowid = max(max_rowid, rowid)
        query = (
            'select x, y, z, w from light where '
            'p = :p and q = :q;'
        )
        rows = self.execute(query, dict(p=p, q=q))
        lights = 0
        for x, y, z, w in rows:
            lights += 1
            packets.append(packet(LIGHT, p, q, x, y, z, w))
        query = (
            'select x, y, z, face, text from sign where '
            'p = :p and q = :q;'
        )
        rows = self.execute(query, dict(p=p, q=q))
        signs = 0
        for x, y, z, face, text in rows:
            signs += 1
            packets.append(packet(SIGN, p, q, x, y, z, face, text))
        if blocks:
            packets.append(packet(KEY, p, q, max_rowid))
        if blocks or lights or signs:
            packets.append(packet(REDRAW, p, q))
        packets.append(packet(CHUNK, p, q))
        client.send_raw(''.join(packets))
    def on_block(self, client, x, y, z, w):
        x, y, z, w = map(int, (x, y, z, w))
        p, q = chunked(x), chunked(z)
        previous = self.get_block(x, y, z)
        message = None
        if AUTH_REQUIRED and client.user_id is None:
            message = 'Only logged in users are allowed to build.'
        elif y <= 0 or y > 255:
            message = 'Invalid block coordinates.'
        elif w not in ALLOWED_ITEMS:
            message = 'That item is not allowed.'
        elif w and previous:
            message = 'Cannot create blocks in a non-empty space.'
        elif not w and not previous:
            message = 'That space is already empty.'
        elif previous in INDESTRUCTIBLE_ITEMS:
            message = 'Cannot destroy that type of block.'
        if message is not None:
            client.send(BLOCK, p, q, x, y, z, previous)
            client.send(REDRAW, p, q)
            client.send(TALK, message)
            return
        query = (
            'insert into block_history (timestamp, user_id, x, y, z, w) '
            'values (:timestamp, :user_id, :x, :y, :z, :w);'
        )
        if RECORD_HISTORY:
            self.execute(query, dict(timestamp=time.time(),
                user_id=client.user_id, x=x, y=y, z=z, w=w))
        query = (
            'insert or replace into block (p, q, x, y, z, w) '
            'values (:p, :q, :x, :y, :z, :w);'
        )
        self.execute(query, dict(p=p, q=q, x=x, y=y, z=z, w=w))
        self.send_block(client, p, q, x, y, z, w)
        for dx in range(-1, 2):
            for dz in range(-1, 2):
                if dx == 0 and dz == 0:
                    continue
                if dx and chunked(x + dx) == p:
                    continue
                if dz and chunked(z + dz) == q:
                    continue
                np, nq = p + dx, q + dz
                self.execute(query, dict(p=np, q=nq, x=x, y=y, z=z, w=-w))
                self.send_block(client, np, nq, x, y, z, -w)
        if w == 0:
            query = (
                'delete from sign where '
                'x = :x and y = :y and z = :z;'
            )
            self.execute(query, dict(x=x, y=y, z=z))
            query = (
                'update light set w = 0 where '
                'x = :x and y = :y and z = :z;'
            )
            self.execute(query, dict(x=x, y=y, z=z))
    def on_light(self, client, x, y, z, w):
        x, y, z, w = map(int, (x, y, z, w))
        p, q = chunked(x), chunked(z)
        block = self.get_block(x, y, z)
        message = None
        if AUTH_REQUIRED and client.user_id is None:
            message = 'Only logged in users are allowed to build.'
        elif block == 0:
            message = 'Lights must be placed on a block.'
        elif w < 0 or w > 15:
            message = 'Invalid light value.'
        if message is not None:
            # TODO: client.send(LIGHT, p, q, x, y, z, previous)
            client.send(REDRAW, p, q)
            client.send(TALK, message)
            return
        query = (
            'insert or replace into light (p, q, x, y, z, w) '
            'values (:p, :q, :x, :y, :z, :w);'
        )
        self.execute(query, dict(p=p, q=q, x=x, y=y, z=z, w=w))
        self.send_light(client, p, q, x, y, z, w)
    def on_sign(self, client, x, y, z, face, *args):
        if AUTH_REQUIRED and client.user_id is None:
            client.send(TALK, 'Only logged in users are allowed to build.')
            return
        text = ','.join(args)
        x, y, z, face = map(int, (x, y, z, face))
        if y <= 0 or y > 255:
            return
        if face < 0 or face > 7:
            return
        if len(text) > 48:
            return
        p, q = chunked(x), chunked(z)
        if text:
            query = (
                'insert or replace into sign (p, q, x, y, z, face, text) '
                'values (:p, :q, :x, :y, :z, :face, :text);'
            )
            self.execute(query,
                dict(p=p, q=q, x=x, y=y, z=z, face=face, text=text))
        else:
            query = (
                'delete from sign where '
                'x = :x and y = :y and z = :z and face = :face;'
            )
            self.execute(query, dict(x=x, y=y, z=z, face=face))
        self.send_sign(client, p, q, x, y, z, face, text)
    def on_position(self, client, x, y, z, rx, ry):
        x, y, z, rx, ry = map(float, (x, y, z, rx, ry))
        client.position = (x, y, z, rx, ry)
        self.send_position(client)
    def on_talk(self, client, *args):
        text = ','.join(args)
        if text.startswith('/'):
            for pattern, func in self.patterns:
                match = pattern.match(text)
                if match:
                    func(client, *match.groups())
                    break
            else:
                client.send(TALK, 'Unrecognized command: "%s"' % text)
        elif text.startswith('@'):
            nick = text[1:].split(' ', 1)[0]
            for other in self.clients:
                if other.nick == nick:
                    client.send(TALK, '%s> %s' % (client.nick, text))
                    other.send(TALK, '%s> %s' % (client.nick, text))
                    break
            else:
                client.send(TALK, 'Unrecognized nick: "%s"' % nick)
        else:
            self.send_talk('%s> %s' % (client.nick, text))
    def on_nick(self, client, nick=None):
        if AUTH_REQUIRED:
            client.send(TALK, 'You cannot change your nick on this server.')
            return
        if nick is None:
            client.send(TALK, 'Your nickname is %s' % client.nick)
        else:
            self.send_talk('%s is now known as %s' % (client.nick, nick))
            client.nick = nick
            self.send_nick(client)
    def on_spawn(self, client):
        client.position = SPAWN_POINT
        client.send(YOU, client.client_id, *client.position)
        self.send_position(client)
    def on_goto(self, client, nick=None):
        if nick is None:
            clients = [x for x in self.clients if x != client]
            other = random.choice(clients) if clients else None
        else:
            nicks = dict((client.nick, client) for client in self.clients)
            other = nicks.get(nick)
        if other:
            client.position = other.position
            client.send(YOU, client.client_id, *client.position)
            self.send_position(client)
    def on_pq(self, client, p, q):
        p, q = map(int, (p, q))
        if abs(p) > 1000 or abs(q) > 1000:
            return
        client.position = (p * CHUNK_SIZE, 0, q * CHUNK_SIZE, 0, 0)
        client.send(YOU, client.client_id, *client.position)
        self.send_position(client)
    def on_help(self, client, topic=None):
        if topic is None:
            client.send(TALK, 'Type "t" to chat. Type "/" to type commands:')
            client.send(TALK, '/goto [NAME], /help [TOPIC], /list, /login NAME, /logout, /nick')
            client.send(TALK, '/offline [FILE], /online HOST [PORT], /pq P Q, /spawn, /view N')
            return
        topic = topic.lower().strip()
        if topic == 'goto':
            client.send(TALK, 'Help: /goto [NAME]')
            client.send(TALK, 'Teleport to another user.')
            client.send(TALK, 'If NAME is unspecified, a random user is chosen.')
        elif topic == 'list':
            client.send(TALK, 'Help: /list')
            client.send(TALK, 'Display a list of connected users.')
        elif topic == 'login':
            client.send(TALK, 'Help: /login NAME')
            client.send(TALK, 'Switch to another registered username.')
            client.send(TALK, 'The login server will be re-contacted. The username is case-sensitive.')
        elif topic == 'logout':
            client.send(TALK, 'Help: /logout')
            client.send(TALK, 'Unauthenticate and become a guest user.')
            client.send(TALK, 'Automatic logins will not occur again until the /login command is re-issued.')
        elif topic == 'offline':
            client.send(TALK, 'Help: /offline [FILE]')
            client.send(TALK, 'Switch to offline mode.')
            client.send(TALK, 'FILE specifies the save file to use and defaults to "craft".')
        elif topic == 'online':
            client.send(TALK, 'Help: /online HOST [PORT]')
            client.send(TALK, 'Connect to the specified server.')
        elif topic == 'nick':
            client.send(TALK, 'Help: /nick [NICK]')
            client.send(TALK, 'Get or set your nickname.')
        elif topic == 'pq':
            client.send(TALK, 'Help: /pq P Q')
            client.send(TALK, 'Teleport to the specified chunk.')
        elif topic == 'spawn':
            client.send(TALK, 'Help: /spawn')
            client.send(TALK, 'Teleport back to the spawn point.')
        elif topic == 'view':
            client.send(TALK, 'Help: /view N')
            client.send(TALK, 'Set viewing distance, 1 - 24.')
    def on_list(self, client):
        client.send(TALK,
            'Players: %s' % ', '.join(x.nick for x in self.clients))
    def send_positions(self, client):
        for other in self.clients:
            if other == client:
                continue
            client.send(POSITION, other.client_id, *other.position)
    def send_position(self, client):
        for other in self.clients:
            if other == client:
                continue
            other.send(POSITION, client.client_id, *client.position)
    def send_nicks(self, client):
        for other in self.clients:
            if other == client:
                continue
            client.send(NICK, other.client_id, other.nick)
    def send_nick(self, client):
        for other in self.clients:
            other.send(NICK, client.client_id, client.nick)
    def send_disconnect(self, client):
        for other in self.clients:
            if other == client:
                continue
            other.send(DISCONNECT, client.client_id)
    def send_block(self, client, p, q, x, y, z, w):
        for other in self.clients:
            if other == client:
                continue
            other.send(BLOCK, p, q, x, y, z, w)
            other.send(REDRAW, p, q)
    def send_light(self, client, p, q, x, y, z, w):
        for other in self.clients:
            if other == client:
                continue
            other.send(LIGHT, p, q, x, y, z, w)
            other.send(REDRAW, p, q)
    def send_sign(self, client, p, q, x, y, z, face, text):
        for other in self.clients:
            if other == client:
                continue
            other.send(SIGN, p, q, x, y, z, face, text)
    def send_talk(self, text):
        log(text)
        for client in self.clients:
            client.send(TALK, text)

def cleanup():
    world = World(None)
    conn = sqlite3.connect(DB_PATH)
    query = 'select x, y, z from block order by rowid desc limit 1;'
    last = list(conn.execute(query))[0]
    query = 'select distinct p, q from block;'
    chunks = list(conn.execute(query))
    count = 0
    total = 0
    delete_query = 'delete from block where x = %d and y = %d and z = %d;'
    print 'begin;'
    for p, q in chunks:
        chunk = world.create_chunk(p, q)
        query = 'select x, y, z, w from block where p = :p and q = :q;'
        rows = conn.execute(query, {'p': p, 'q': q})
        for x, y, z, w in rows:
            if chunked(x) != p or chunked(z) != q:
                continue
            total += 1
            if (x, y, z) == last:
                continue
            original = chunk.get((x, y, z), 0)
            if w == original or original in INDESTRUCTIBLE_ITEMS:
                count += 1
                print delete_query % (x, y, z)
    conn.close()
    print 'commit;'
    print >> sys.stderr, '%d of %d blocks will be cleaned up' % (count, total)

def main():
    if len(sys.argv) == 2 and sys.argv[1] == 'cleanup':
        cleanup()
        return
    host, port = DEFAULT_HOST, DEFAULT_PORT
    if len(sys.argv) > 1:
        host = sys.argv[1]
    if len(sys.argv) > 2:
        port = int(sys.argv[2])
    log('SERV', host, port)
    model = Model(None)
    model.start()
    server = Server((host, port), Handler)
    server.model = model
    server.serve_forever()

if __name__ == '__main__':
    main()

# gcc -std=c99 -O3 -shared -o world \
#   -I src -I deps/noise deps/noise/noise.c src/world.c

from ctypes import CDLL, CFUNCTYPE, c_float, c_int, c_void_p
from collections import OrderedDict

dll = CDLL('./world')

WORLD_FUNC = CFUNCTYPE(None, c_int, c_int, c_int, c_int, c_void_p)

def dll_seed(x):
    dll.seed(x)

def dll_create_world(p, q):
    result = {}
    def world_func(x, y, z, w, arg):
        result[(x, y, z)] = w
    dll.create_world(p, q, WORLD_FUNC(world_func), None)
    return result

dll.simplex2.restype = c_float
dll.simplex2.argtypes = [c_float, c_float, c_int, c_float, c_float]
def dll_simplex2(x, y, octaves=1, persistence=0.5, lacunarity=2.0):
    return dll.simplex2(x, y, octaves, persistence, lacunarity)

dll.simplex3.restype = c_float
dll.simplex3.argtypes = [c_float, c_float, c_float, c_int, c_float, c_float]
def dll_simplex3(x, y, z, octaves=1, persistence=0.5, lacunarity=2.0):
    return dll.simplex3(x, y, z, octaves, persistence, lacunarity)

class World(object):
    def __init__(self, seed=None, cache_size=64):
        self.seed = seed
        self.cache = OrderedDict()
        self.cache_size = cache_size
    def create_chunk(self, p, q):
        if self.seed is not None:
            dll_seed(self.seed)
        return dll_create_world(p, q)
    def get_chunk(self, p, q):
        try:
            chunk = self.cache.pop((p, q))
        except KeyError:
            chunk = self.create_chunk(p, q)
        self.cache[(p, q)] = chunk
        if len(self.cache) > self.cache_size:
            self.cache.popitem(False)
        return chunk

.DS_Store
Makefile
CMakeCache.txt
CMakeFiles
cmake_install.cmake
config.py
craft
world
env
log.txt
*.o
*.db
*.exe
*.dll
*.pyc


# This file allows you to programmatically create blocks in Craft.
# Please use this wisely. Test on your own server first. Do not abuse it.

import requests
import socket
import sqlite3
import sys

DEFAULT_HOST = '127.0.0.1'
DEFAULT_PORT = 4080

EMPTY = 0
GRASS = 1
SAND = 2
STONE = 3
BRICK = 4
WOOD = 5
CEMENT = 6
DIRT = 7
PLANK = 8
SNOW = 9
GLASS = 10
COBBLE = 11
LIGHT_STONE = 12
DARK_STONE = 13
CHEST = 14
LEAVES = 15
CLOUD = 16
TALL_GRASS = 17
YELLOW_FLOWER = 18
RED_FLOWER = 19
PURPLE_FLOWER = 20
SUN_FLOWER = 21
WHITE_FLOWER = 22
BLUE_FLOWER = 23

OFFSETS = [
    (-0.5, -0.5, -0.5),
    (-0.5, -0.5, 0.5),
    (-0.5, 0.5, -0.5),
    (-0.5, 0.5, 0.5),
    (0.5, -0.5, -0.5),
    (0.5, -0.5, 0.5),
    (0.5, 0.5, -0.5),
    (0.5, 0.5, 0.5),
]

def sphere(cx, cy, cz, r, fill=False, fx=False, fy=False, fz=False):
    result = set()
    for x in range(cx - r, cx + r + 1):
        if fx and x != cx:
            continue
        for y in range(cy - r, cy + r + 1):
            # if y < cy:
            #     continue # top hemisphere only
            if fy and y != cy:
                continue
            for z in range(cz - r, cz + r + 1):
                if fz and z != cz:
                    continue
                inside = False
                outside = fill
                for dx, dy, dz in OFFSETS:
                    ox, oy, oz = x + dx, y + dy, z + dz
                    d2 = (ox - cx) ** 2 + (oy - cy) ** 2 + (oz - cz) ** 2
                    d = d2 ** 0.5
                    if d < r:
                        inside = True
                    else:
                        outside = True
                if inside and outside:
                    result.add((x, y, z))
    return result

def circle_x(x, y, z, r, fill=False):
    return sphere(x, y, z, r, fill, fx=True)

def circle_y(x, y, z, r, fill=False):
    return sphere(x, y, z, r, fill, fy=True)

def circle_z(x, y, z, r, fill=False):
    return sphere(x, y, z, r, fill, fz=True)

def cylinder_x(x1, x2, y, z, r, fill=False):
    x1, x2 = sorted((x1, x2))
    result = set()
    for x in range(x1, x2 + 1):
        result |= circle_x(x, y, z, r, fill)
    return result

def cylinder_y(x, y1, y2, z, r, fill=False):
    y1, y2 = sorted((y1, y2))
    result = set()
    for y in range(y1, y2 + 1):
        result |= circle_y(x, y, z, r, fill)
    return result

def cylinder_z(x, y, z1, z2, r, fill=False):
    z1, z2 = sorted((z1, z2))
    result = set()
    for z in range(z1, z2 + 1):
        result |= circle_z(x, y, z, r, fill)
    return result

def cuboid(x1, x2, y1, y2, z1, z2, fill=True):
    x1, x2 = sorted((x1, x2))
    y1, y2 = sorted((y1, y2))
    z1, z2 = sorted((z1, z2))
    result = set()
    a = (x1 == x2) + (y1 == y2) + (z1 == z2)
    for x in range(x1, x2 + 1):
        for y in range(y1, y2 + 1):
            for z in range(z1, z2 + 1):
                n = 0
                n += x in (x1, x2)
                n += y in (y1, y2)
                n += z in (z1, z2)
                if not fill and n <= a:
                    continue
                result.add((x, y, z))
    return result

def pyramid(x1, x2, y, z1, z2, fill=False):
    x1, x2 = sorted((x1, x2))
    z1, z2 = sorted((z1, z2))
    result = set()
    while x2 >= x1 and z2 >= z2:
        result |= cuboid(x1, x2, y, y, z1, z2, fill)
        y, x1, x2, z1, z2 = y + 1, x1 + 1, x2 - 1, z1 + 1, z2 - 1
    return result

def get_identity():
    query = (
        'select username, token from identity_token where selected = 1;'
    )
    conn = sqlite3.connect('auth.db')
    rows = conn.execute(query)
    for row in rows:
        return row
    raise Exception('No identities found.')

class Client(object):
    def __init__(self, host, port):
        self.conn = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
        self.conn.connect((host, port))
        self.authenticate()
    def authenticate(self):
        username, identity_token = get_identity()
        url = 'https://craft.michaelfogleman.com/api/1/identity'
        payload = {
            'username': username,
            'identity_token': identity_token,
        }
        response = requests.post(url, data=payload)
        if response.status_code == 200 and response.text.isalnum():
            access_token = response.text
            self.conn.sendall('A,%s,%s\n' % (username, access_token))
        else:
            raise Exception('Failed to authenticate.')
    def set_block(self, x, y, z, w):
        self.conn.sendall('B,%d,%d,%d,%d\n' % (x, y, z, w))
    def set_blocks(self, blocks, w):
        key = lambda block: (block[1], block[0], block[2])
        for x, y, z in sorted(blocks, key=key):
            self.set_block(x, y, z, w)
    def bitmap(self, sx, sy, sz, d1, d2, data, lookup):
        x, y, z = sx, sy, sz
        dx1, dy1, dz1 = d1
        dx2, dy2, dz2 = d2
        for row in data:
            x = sx if dx1 else x
            y = sy if dy1 else y
            z = sz if dz1 else z
            for c in row:
                w = lookup.get(c)
                if w is not None:
                    self.set_block(x, y, z, w)
                x, y, z = x + dx1, y + dy1, z + dz1
            x, y, z = x + dx2, y + dy2, z + dz2

def get_client():
    default_args = [DEFAULT_HOST, DEFAULT_PORT]
    args = sys.argv[1:] + [None] * len(default_args)
    host, port = [a or b for a, b in zip(args, default_args)]
    client = Client(host, int(port))
    return client

def main():
    client = get_client()
    set_block = client.set_block
    set_blocks = client.set_blocks
    # set_blocks(circle_y(0, 32, 0, 16, True), STONE)
    # set_blocks(circle_y(0, 33, 0, 16), BRICK)
    # set_blocks(cuboid(-1, 1, 1, 31, -1, 1), CEMENT)
    # set_blocks(cuboid(-1024, 1024, 32, 32, -3, 3), STONE)
    # set_blocks(cuboid(-3, 3, 32, 32, -1024, 1024), STONE)
    # set_blocks(cuboid(-1024, 1024, 33, 33, -3, -3), BRICK)
    # set_blocks(cuboid(-1024, 1024, 33, 33, 3, 3), BRICK)
    # set_blocks(cuboid(-3, -3, 33, 33, -1024, 1024), BRICK)
    # set_blocks(cuboid(3, 3, 33, 33, -1024, 1024), BRICK)
    # set_blocks(sphere(0, 32, 0, 16), GLASS)
    # for y in range(1, 32):
    #     set_blocks(circle_y(0, y, 0, 4, True), CEMENT)
    # set_blocks(circle_x(16, 33, 0, 3), BRICK)
    # set_blocks(circle_x(-16, 33, 0, 3), BRICK)
    # set_blocks(circle_z(0, 33, 16, 3), BRICK)
    # set_blocks(circle_z(0, 33, -16, 3), BRICK)
    # for x in range(0, 1024, 32):
    #     set_blocks(cuboid(x - 1, x + 1, 31, 32, -1, 1), CEMENT)
    #     set_blocks(cuboid(-x - 1, -x + 1, 31, 32, -1, 1), CEMENT)
    #     set_blocks(cuboid(x, x, 1, 32, -1, 1), CEMENT)
    #     set_blocks(cuboid(-x, -x, 1, 32, -1, 1), CEMENT)
    # for z in range(0, 1024, 32):
    #     set_blocks(cuboid(-1, 1, 31, 32, z - 1, z + 1), CEMENT)
    #     set_blocks(cuboid(-1, 1, 31, 32, -z - 1, -z + 1), CEMENT)
    #     set_blocks(cuboid(-1, 1, 1, 32, z, z), CEMENT)
    #     set_blocks(cuboid(-1, 1, 1, 32, -z, -z), CEMENT)
    # for x in range(0, 1024, 8):
    #     set_block(x, 32, 0, CEMENT)
    #     set_block(-x, 32, 0, CEMENT)
    # for z in range(0, 1024, 8):
    #     set_block(0, 32, z, CEMENT)
    #     set_block(0, 32, -z, CEMENT)
    # set_blocks(pyramid(32, 32+64-1, 12, 32, 32+64-1), COBBLE)
    # outer = circle_y(0, 11, 0, 176 + 3, True)
    # inner = circle_y(0, 11, 0, 176 - 3, True)
    # set_blocks(outer - inner, STONE)
    # a = sphere(-32, 48, -32, 24, True)
    # b = sphere(-24, 40, -24, 24, True)
    # set_blocks(a - b, PLANK)
    # set_blocks(cylinder_x(-64, 64, 32, 0, 8), STONE)
    # data = [
    #     '...............................',
    #     '..xxx..xxxx...xxx..xxxxx.xxxxx.',
    #     '.x...x.x...x.x...x.x.......x...',
    #     '.x.....xxxx..xxxxx.xxx.....x...',
    #     '.x...x.x..x..x...x.x.......x...',
    #     '..xxx..x...x.x...x.x.......x...',
    #     '...............................',
    # ]
    # lookup = {
    #     'x': STONE,
    #     '.': PLANK,
    # }
    # client.bitmap(0, 32, 32, (1, 0, 0), (0, -1, 0), data, lookup)

if __name__ == '__main__':
    main()

cmake_minimum_required(VERSION 2.8)

project(craft)

FILE(GLOB SOURCE_FILES src/*.c)

add_executable(
    craft
    ${SOURCE_FILES}
    deps/glew/src/glew.c
    deps/lodepng/lodepng.c
    deps/noise/noise.c
    deps/sqlite/sqlite3.c
    deps/tinycthread/tinycthread.c)

add_definitions(-std=c99 -O3)

add_subdirectory(deps/glfw)
include_directories(deps/glew/include)
include_directories(deps/glfw/include)
include_directories(deps/lodepng)
include_directories(deps/noise)
include_directories(deps/sqlite)
include_directories(deps/tinycthread)

if(MINGW)
    set(CMAKE_LIBRARY_PATH ${CMAKE_LIBRARY_PATH}
        "C:/Program Files/CURL/lib" "C:/Program Files (x86)/CURL/lib")
    set(CMAKE_INCLUDE_PATH ${CMAKE_INCLUDE_PATH}
        "C:/Program Files/CURL/include" "C:/Program Files (x86)/CURL/include")
endif()

find_package(CURL REQUIRED)
include_directories(${CURL_INCLUDE_DIR})

if(APPLE)
    target_link_libraries(craft glfw
        ${GLFW_LIBRARIES} ${CURL_LIBRARIES})
endif()

if(UNIX)
    target_link_libraries(craft dl glfw
        ${GLFW_LIBRARIES} ${CURL_LIBRARIES})
endif()

if(MINGW)
    target_link_libraries(craft ws2_32.lib glfw
        ${GLFW_LIBRARIES} ${CURL_LIBRARIES})
endif()
