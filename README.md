# Databrary Changes

This documentation outlines changes made to [scs](https://github.com/databrary/scs) in order to support
persistent cookies and setting prefix per cookie.

## Persistent Cookies

These extensions enable setting persistent cookies on a per cookie basis (this is to support "remember me").

This sets an expiry of one year on the session.

```go
func SetPersist(r *http.Request, persist bool) error {
    s, err := sessionFromContext(r)
    if err != nil {
        return err
    }
    s.opts.persist = persist
    if persist {
        s.deadline = time.Now().AddDate(1, 0, 0)
    }
    return nil
}
```

This function needs to be called after creating the cookie.

This sync the session with the `persist` setting on the cookie.

```go
func load(r *http.Request, engine Engine, opts *options) (*http.Request, error) {
    cookie, err := r.Cookie(CookieName)
    if err == http.ErrNoCookie {
        return newSession(r, engine, opts)
    } else if err != nil {
        return nil, err
    }

    if cookie.Value == "" {
        return newSession(r, engine, opts)
    }
    token := cookie.Value

    j, found, err := engine.Find(token)
    if err != nil {
        return nil, err
    }
    if found == false {
        return newSession(r, engine, opts)
    }
    //>>>>>
    data, deadline, persist, err := decodeFromJSON(j)
    //<<<<<
    if err != nil {
        return nil, err
    }
    //>>>>>
    opts.persist = persist
    if persist {
        deadline = time.Now().AddDate(1, 0, 0)
    }
    //<<<<<
    s := &session{
        token:    token,
        data:     data,
        deadline: deadline,
        engine:   engine,
        opts:     opts,
    }

    return requestWithSession(r, s), nil
}
```


These serialize/deserialize the sessions with the `persist` flag as well.

```go
func encodeToJSON(data map[string]interface{}, deadline time.Time, persist bool) ([]byte, error) {
    return json.Marshal(&struct {
        Data     map[string]interface{} `json:"data"`
        Deadline int64                  `json:"deadline"`
        Persist  bool                   `json:"persist"`
    }{
        Data:     data,
        Deadline: deadline.UnixNano(),
        Persist:  persist,
    })
}

func decodeFromJSON(j []byte) (map[string]interface{}, time.Time, bool, error) {
    aux := struct {
        Data     map[string]interface{} `json:"data"`
        Deadline int64                  `json:"deadline"`
        Persist  bool                   `json:"persist"`
    }{}

    dec := json.NewDecoder(bytes.NewReader(j))
    dec.UseNumber()
    err := dec.Decode(&aux)
    if err != nil {
        return nil, time.Time{}, false, err
    }
    return aux.Data, time.Unix(0, aux.Deadline), aux.Persist, nil
}

```