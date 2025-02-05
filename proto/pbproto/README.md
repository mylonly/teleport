## pbproto

pbproto is implemented PTOTOBUF socket communication protocol.

### Message Bytes

`{length bytes}` `{xferPipe length byte}` `{xferPipe bytes}` `{protobuf bytes}`

- `{length bytes}`: uint32, 4 bytes, big endian
- `{xferPipe length byte}`: 1 byte
- `{xferPipe bytes}`: one byte one xfer
- `{protobuf bytes}`: {"seq":%d,"mtype":%d,"serviceMethod":%q,"meta":%q,"bodyCodec":%d,"body":"%s"}

### Usage

`import "github.com/mylonly/teleport/proto/pbproto"`

#### Test

```go
package pbproto_test

import (
	"testing"
	"time"

	tp "github.com/mylonly/teleport"
	"github.com/mylonly/teleport/proto/pbproto"
	"github.com/mylonly/teleport/xfer/gzip"
)

type Home struct {
	tp.CallCtx
}

func (h *Home) Test(arg *map[string]string) (map[string]interface{}, *tp.Rerror) {
	h.Session().Push("/push/test", map[string]string{
		"your_id": string(h.PeekMeta("peer_id")),
	})
	return map[string]interface{}{
		"arg": *arg,
	}, nil
}

func TestPbProto(t *testing.T) {
	gzip.Reg('g', "gizp-5", 5)

	// server
	srv := tp.NewPeer(tp.PeerConfig{ListenPort: 9090})
	srv.RouteCall(new(Home))
	go srv.ListenAndServe(pbproto.NewPbProtoFunc())
	time.Sleep(1e9)

	// client
	cli := tp.NewPeer(tp.PeerConfig{})
	cli.RoutePush(new(Push))
	sess, err := cli.Dial(":9090", pbproto.NewPbProtoFunc())
	if err != nil {
		t.Error(err)
	}
	var result interface{}
	rerr := sess.Call("/home/test",
		map[string]string{
			"author": "henrylee2cn",
		},
		&result,
		tp.WithAddMeta("peer_id", "110"),
		tp.WithXferPipe('g'),
	).Rerror()
	if rerr != nil {
		t.Error(rerr)
	}
	t.Logf("result:%v", result)
	time.Sleep(3e9)
}

type Push struct {
	tp.PushCtx
}

func (p *Push) Test(arg *map[string]string) *tp.Rerror {
	tp.Infof("receive push(%s):\narg: %#v\n", p.IP(), arg)
	return nil
}
```

test command:

```sh
go test -v -run=TestPbProto
```
