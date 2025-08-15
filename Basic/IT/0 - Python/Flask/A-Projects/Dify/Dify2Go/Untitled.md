```bash
go mod init dify2go
```

一个非常流行的配置包

```bash
go get github.com/spf13/viper
```

```go
package config

import "github.com/spf13/viper"

type Config struct {
	DBDSN      string
	RedisAddr  string
	RedisDB    int
	DSLMaxSize int64
}

var Cfg Config

func Init() {
	viper.SetDefault("DB_DSN", "postgres://user:pass@localhost:5432/dify")
	viper.SetDefault("REDIS_ADDR", "localhost:6379")
	viper.SetDefault("REDIS_DB", 0)
	viper.SetDefault("DSL_MAX_SIZE", 10*1024*1024)
	viper.AutomaticEnv()

	Cfg = Config{
		DBDSN:      viper.GetString("DB_DSN"),
		RedisAddr:  viper.GetString("REDIS_ADDR"),
		RedisDB:    viper.GetInt("REDIS_DB"),
		DSLMaxSize: viper.GetInt64("DSL_MAX_SIZE"),
	}
}
```

```bash
go get gorm.io/gorm gorm.io/driver/postgres
```

