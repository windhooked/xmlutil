# xmlutil
https://stackoverflow.com/questions/11767642/wsdl-soap-support-on-go

There isn't support for WSDL in Go. Support in other languages are either static or dynamic: Either structs are pre-generated from the WSDL, or it's done on the fly with hash tables.

You can, however, encode and decode SOAP requests manually. I found that the standard encoding/xml package to be insufficient for SOAP. There are so many quirks in different servers, and the limitations in encoding/xml make it difficult generate a request these servers are happy with.

For example, some servers need xsi:type="xsd:string" on every string tag. In order to do this properly your struct needs to look like this for encoding/xml:

type MethodCall struct {
    One XSI
    Two XSI
}

type XSI struct {
    Type string `xml:"xsi:type,attr"`
    Vaue string `xml:",chardata"`
}
And you construct it like this:

MethodCall{
    XSI{"xsd:string", "One"},
    XSI{"xsd:string", "Two"},
}
Which gives you:

<MethodCall>
    <One xsi:type="xsd:string">One</One>
    <Two xsi:type="xsd:string">Two</Two>
</MethodCall>
Now this might be ok. It certainly gets the job done. But what if you needed more than just a string? encoding/xml currently doesn't support interface{}.

As you can see this gets complicated. If you had one SOAP API to integrate, this probably wouldn't be too bad. What if you had several, each with their own quirks?

Wouldn't it be nice if you could just do this?

type MethodCall struct {
    One string
    Two string
}
Then say to encoding/xml: "This server want xsi types".

To solve this problem I created github.com/webconnex/xmlutil. It's a work in progress. It doesn't have all the features of encoding/xml's encoder/decoder, but it has what is needed for SOAP.

Here's a working example:

package main

import (
    "bytes"
    "encoding/xml"
    "fmt"
    "github.com/webconnex/xmlutil"
    "log"
    //"net/http"
)

type Envelope struct {
    Body `xml:"soap:"`
}

type Body struct {
    Msg interface{}
}

type MethodCall struct {
    One string
    Two string
}

type MethodCallResponse struct {
    Three string
}

func main() {
    x := xmlutil.NewXmlUtil()
    x.RegisterNamespace("http://www.w3.org/2001/XMLSchema-instance", "xsi")
    x.RegisterNamespace("http://www.w3.org/2001/XMLSchema", "xsd")
    x.RegisterNamespace("http://www.w3.org/2003/05/soap-envelope", "soap")
    x.RegisterTypeMore(Envelope{}, xml.Name{"http://www.w3.org/2003/05/soap-envelope", ""},
        []xml.Attr{
            xml.Attr{xml.Name{"xmlns", "xsi"}, "http://www.w3.org/2001/XMLSchema-instance"},
            xml.Attr{xml.Name{"xmlns", "xsd"}, "http://www.w3.org/2001/XMLSchema"},
            xml.Attr{xml.Name{"xmlns", "soap"}, "http://www.w3.org/2003/05/soap-envelope"},
        })
    x.RegisterTypeMore("", xml.Name{}, []xml.Attr{
        xml.Attr{xml.Name{"http://www.w3.org/2001/XMLSchema-instance", "type"}, "xsd:string"},
    })

    buf := new(bytes.Buffer)
    buf.WriteString(`<?xml version="1.0" encoding="utf-8"?>`)
    buf.WriteByte('\n')
    enc := x.NewEncoder(buf)
    env := &Envelope{Body{MethodCall{
        One: "one",
        Two: "two",
    }}}
    if err := enc.Encode(env); err != nil {
        log.Fatal(err)
    }
    // Print request
    bs := buf.Bytes()
    bs = bytes.Replace(bs, []byte{'>', '<'}, []byte{'>', '\n', '<'}, -1)
    fmt.Printf("%s\n\n", bs)

    /*
        // Send response, SOAP 1.2, fill in url, namespace, and action
        var r *http.Response
        if r, err = http.Post(url, "application/soap+xml; charset=utf-8; action="+namespace+"/"+action, buf); err != nil {
            return
        }
        dec := x.NewDecoder(r.Body)
    */
    // Decode response
    dec := x.NewDecoder(bytes.NewBufferString(`<?xml version="1.0" encoding="utf-8"?>
    <soap:Envelope>
        <soap:Body>
            <MethodCallResponse>
                <Three>three</Three>
            </MethodCallResponse>
        </soap:Body>
    </soap:Envelope>`))
    find := []xml.Name{
        xml.Name{"", "MethodCallResponse"},
        xml.Name{"http://www.w3.org/2003/05/soap-envelope", "Fault"},
    }
    var start *xml.StartElement
    var err error
    if start, err = dec.Find(find); err != nil {
        log.Fatal(err)
    }
    if start.Name.Local == "Fault" {
        log.Fatal("Fault!") // Here you can decode a Soap Fault
    }
    var resp MethodCallResponse
    if err := dec.DecodeElement(&resp, start); err != nil {
        log.Fatal(err)
    }
    fmt.Printf("%#v\n\n", resp)
}
With the above example I use the Find method to get the response object, or a Fault. This isn't strictly necessary. You can also do it like this:

x.RegisterType(MethodCallResponse{})
...
// Decode response
dec := x.NewDecoder(bytes.NewBufferString(`<?xml version="1.0" encoding="utf-8"?>
<soap:Envelope>
    <soap:Body>
        <MethodCallResponse>
            <Three>three</Three>
        </MethodCallResponse>
    </soap:Body>
</soap:Envelope>`))
var start *xml.StartElement
var resp Envelope
if err := dec.DecodeElement(&resp, start); err != nil {
    log.Fatal(err)
}
fmt.Printf("%#v\n\n", resp)
You'll find the Find method useful when your data looks like this:

<soap:Envelope>
  <soap:Body>
    <MethodResponse>
      <MethodResult>
        <diffgr:diffgram>
          <NewDataSet>
            <Table1 diffgr:id="Table1" msdata:rowOrder="0" diffgr:hasChanges="inserted">
              <Three>three</Three>
            </Table1>
          </NewDataSet>
        </diffgr:diffgram>
      </MethodResult>
    </MethodResponse>
  </soap:Body>
</soap:Envelope>
This is a DiffGram, part of Microsoft .NET. You can use the Find method to get to Table1. The Decode and DecodeElement method also works on slices. So you can pass in a []MethodCallResponse if NewDataSet happens to contain more than one result.

I do agree with Zippower that SOAP does suck. But unfortunately a lot of enterprises use SOAP, and you're sometimes forced to use these APIs. With the xmlutil package I hope to make it a little less painful to work with.
