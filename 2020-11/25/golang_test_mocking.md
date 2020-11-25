# Golang의 test mocking에 관해서

Golang's Mocking Techniques - Kyle Yost을 보고 알아들은 점 정리

하나의 기능에 대해서 테스트를 만들고 여러가지 경우를 subtest로 만들어서 struct에 넣어주고 range로 돌리면서 하나씩 t.Run을 사용함. 어떤 서브테스트에서 문제가 발생했는지 알기 위함임.

gomock, go-sqlmock, testify, mockery 등에 대해서 간단한 소개를 진행함.

## Higher order Function을 만들어서 테스트시 장점과 단점

* package level function을 테스트하는데 사용한다.

테스트를 원하는 함수중 package 내부 함수에 대해 mocking을 진행한다. \
OpenDB라는 함수를 테스트 하고 싶은데, OpenDB 내부에서 사용하는 패키지 제공 함수인 sql.open이라는 함수에 대해서 동작을 모킹해야히면 sql.open과 input, output type이 같은 함수형 type을 만든다. \
이후에는 sql.open-mock의 반환값을 다르게 정의하며 테스트를 원하는 함수의 인자로 활용할 수 있다.

* 장점
  * 간단하고 stateless한 테스트를 진행할 수 있다.

* 단점
  * parameter를 리스트의 형태로 주는것이 좀 지저분해 보일 수 있다.

## Monkey Patching으로 테스트시 장점과 단점

* package level function을 테스트하는데 사용한다.

high order function을 사용하는것과 비슷하다. 차이점은 mocking이 필요한 package 함수를 테스트하는 함수에 인자로 주지 않고 var로 함수 내부 상단에 정의해놓는다는 것이다. \

* 장점
  * function에 대한 mocking 없이 끌어다가 사용할 수 있다.

* 단점
  * var의 형태로 테스트를 진행하는 함수 내부에 존재하는 것임으로 state를 가진다. 만약 테스트에서 이 인자에 대해 변형이 일어날 경우에 영향을 미칠 수 있다.
  * 해당 이유로 parallelism에 악영향을 미친다.

## Interface Substitution

* concrete type의 method에 대한 mocking이 필요할때 쓰인다.

I/O 패키지에 포함된 Interface ReadCloser는 Interface Reader, Interface Closer를 가지고 있다. \
이의 동작을 모킹하기 위해서는 struct mockReadCloser를 만들고 내부의 함수들을 mocking 하면 된다. \

```golang
type (
    mockReadCloser struct {
        expectedData []byte
        expectedErr  error
    }
)

// allow subtests to fill out expectedData and expectedErr field
func (mrc *mockReadCloser) Read(p []byte) (n int, err error) {
    copy(p, mrc.expectedData)
    return 0, mrc.expectedErr
}
// do nothing on close
func (mrc *mockReadCloser) Close() error { return nil }
```

위와 같이 mocking하고 mockReadCloser.expectedData, mockReadCloser.expectedErr에 데이터를 넣어주면, 우리가 테스트를 진행하는 함수에서 Read, Close 등을 부르고 값을 반환할 것이다. \
이후에는 test 함수의 반환값과 예상값을 비교하면 된다.

* 장점
  * Accept Interface, return struct의 간단한 구조로 동작을 mocking 가능

## Embedding Interfaces

* large interface에 정의된 small set of methods를 mocking 할 때 사용한다.

DynamoDB가 예제로 사용된다. <https://docs.aws.amazon.com/sdk-for-go/api/service/dynamodb/dynamodbiface/>

예제 코드

```golang
// myFunc uses an SDK service client to make a request to
// Amazon DynamoDB.
func myFunc(svc dynamodbiface.DynamoDBAPI) bool {
    // Make svc.BatchGetItem request
}
func main() {
    sess := session.New()
    svc := dynamodb.New(sess)
    myFunc(svc)
}
```

myFunc가 DynamoDB의 모든 API를 포함하는 dynamodbiface를 가진다. 우리는 BatchGetItem에 대한 mocking만 필요한데도.

```golang
// Define a mock struct to be used in your unit tests of myFunc.
type mockDynamoDBClient struct {
    dynamodbiface.DynamoDBAPI
}
func (m *mockDynamoDBClient) BatchGetItem(input *dynamodb.BatchGetItemInput) (*dynamodb.BatchGetItemOutput, error) {
    // mock response/functionality
}
func TestMyFunc(t *testing.T) {
    // Setup Test
    mockSvc := &mockDynamoDBClient{}
    myfunc(mockSvc)
    // Verify myFunc's functionality
}
```

wrapping하는 mock struct를 만들고 해당 struct에 함수를 정의함으로 해결. 간단하네

## Mocking HTTP Calls

* HTTP call을 모킹해야 하는 경우

test는 되도록이면 external service에 연결되지 말아야 한다. `net/http/httptest`라는 패키지가 local loopback interface를 제공해준다. external interface없이 정해진 internal http input에 대해 정해진 output을 반환해주는 것이다.

```golang
type Response struct {
    ID          int    `json:"id"`
    Name        string `json:"name"`
    Description string `json:"description"`
}

func MakeHTTPCall(url string) (*Response, error) {
    resp, err := http.Get(url)
    if err != nil {
        return nil, err
    }
    body, err := ioutil.ReadAll(resp.Body)
    if err != nil {
        return nil, err
    }
    r := &Response{}
    if err := json.Unmarshal(body, r); err != nil {
        return nil, err
    }
    return r, nil
}
```

이렇게 선언하면 url을 받아서 response를 리턴하는 형태의 기초적인 함수가 되는 것이다.

```golang
func TestMakeHTTPCall(t *testing.T) {
    testTable := []struct {
        name             string
        server           *httptest.Server
        expectedResponse *Response
        expectedErr      error
    }{
        {
            name: "happy-server-response",
            server: httptest.NewServer(http.HandlerFunc(func(w http.ResponseWriter, r *http.Request) {
                w.WriteHeader(http.StatusOK)
                w.Write([]byte(`{"id": 1, "name": "kyle", "description": "novice gopher"}`))
            })),
            expectedResponse: &Response{
                ID:          1,
                Name:        "kyle",
                Description: "novice gopher",
            },
            expectedErr: nil,
        },
    }
    for _, tc := range testTable {
        t.Run(tc.name, func(t *testing.T) {
            defer tc.server.Close()
            resp, err := MakeHTTPCall(tc.server.URL)
            if !reflect.DeepEqual(resp, tc.expectedResponse) {
                t.Errorf("expected (%v), got (%v)", tc.expectedResponse, resp)
            }
            if !errors.Is(err, tc.expectedErr) {
                t.Errorf("expected (%v), got (%v)", tc.expectedErr, err)
            }
        })
    }
}
```

httptest 패키지에 포함된 Server를 정의 후, response도 정의해놓는다. 아래쪽 함수에서 MakeHTTPCall로 server에서 write하는 데이터를 response에 넣어서 반환받는다. 이를 비교하면 끝
