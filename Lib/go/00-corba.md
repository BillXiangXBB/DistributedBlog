# corba

## corba简介
corba是spf13提供的一个可用来创建强大的CLI命令行命令的库；
corba也是一个应用程序，它将生成应用程序脚手架以快速开发基于corba的应用程序。
corba主要提供以下功能：
* 简单的基于子命令的CLI，如app server，app fetch等
* 与POSIX完全兼容的标志（包括长短版本）
* 嵌套子命令
* 全局、局部和串联的标志
* 通过corba init appname和corba add cmdname可简单的生成应用和命令行
* 智能提醒（如输入app srver...提示你是想使用app server）
* 为命令和标志自动生成帮助
* 自动识别如-h，--help等帮助标志
* 为应用程序自动生成shell自动补全（bash, zsh, fish, powershell）
* 为应用程序自动生成使用手册页
* 使用命令别名，可在不修改原有内容的基础上修改命令
* 灵活定义自己的帮助、使用等
* 对于12要素应用，集成viper

## corba基本概念
### Commands（命令）
Commands代表操作，corba定义了Command数据结构来表示一条命令。其数据结构包含如下属性：
* Use：string，命令用法
* Aliases：[]string，命令别名数组
* SuggestFor：[]string，建议作为命令名称的数组，类似别名，但只是建议
* Short：string，简短的帮助内容，使用help输出
* Long：string，详细的帮助内容，使用help <this-command>输出
* Example：string，使用该命令的示例
* ValidArgs：[]string，bash自动补全可接受的所有有效非标志参数
* ValidArgsFunction：func(cmd *Command, args []string, toComplete []string) ([]string, ShellCompDirective)，用于判断在bash自动补全命令中有效的非标志参数，ValidArgs和ValidArgsFunction只能二选一
* Args：PositionalArgs，命令行参数
* ArgAliases：[]string，参数别名
* BashCompletionFunction：string，bash自动补全生成器使用的函数
* Deprecated：string，如果命令废弃，输出该字段
* Hidden：bool，如果命令隐藏，并不会在可用命令中显示则设置该字段
* Annotations：map[string]string，应用可以基于该字段来识别或分组命令
* Version：string，命令行版本，如果指定该字段，则可使用--version或-v显示命令行版本
* PersistentPreRun：func(cmd *Command, args []string)
* PersistentPreRunE：func(cmd *Command, args []string) error
* PreRun：func(cmd *Command, args []string)
* PreRunE：func(cmd *Command, args []string) error
* Run：func(cmd *Command, args []string)
* RunE：func(cmd *Command, args []string) error
* PostRun：func(cmd *Command, args []string)
* PostRunE：func(cmd *Command, args []string) error
* PersistentPostRun：func(cmd *Command, args []string)
* PersistentPostRunE: func(cmd *Command, args []string) error
* SilenceErrors：bool，消除下游错误的选项
* SilenceUsage：bool，当发生错误时，不返回用法的选项
* DisableFlagParsing：bool，如果该值设置为true，所有Flag标签都将作为参数传递给命令行
* DisableAutoGenTag：bool，表示是否自动生成文档
* DisableFlagsInUseLine：bool，当打印帮助和生成文档时，禁止在命令的用法中打印额外的flags
* DisableSuggestions：bool，禁止打印与"unknown command"消息一起的基于编辑距离的建议
* SuggestionsMinimumDistance：int，显示建议的最小编辑距离，必须大于0
* TraverseChildren：bool，执行子命令时先解析所有父命令的标志
* FParseErrWhitelist：FParseErrWhitelist，可忽略的标志解析错误
* ctx：context.Context，命令行上下文
* commands：[]*Command，该程序支持的所有命令列表
* parent：*Command，该命令的父命令
* commandsMaxUseLen：int，用于填充用法的命令字符串的最大长度
* commandsMaxCommandPathLen：int，用于填充命令路径的命令字符串的最大长度
* commandsMaxNameLen：int，用于填充名称的命令字符串的最大长度
* commandsAreSorted：bool，命令是否排序
* commandCalledAs：struct {name string, called bool}，用于调用该命令的名称与别名值
* args：[]string，从flags中解析的参数
* flagErrorBuf：*bytes.Buffer，保存pflag中的所有错误信息
* flags：*flag.FlagSet，包含所有flags的列表
* pflags：*flag.FlagSet，包含所有全局flags的列表
* lflags：*flag.FlagSet，包含所有局部flags的列表
* iflags：*flag.FlagSet，包含所有继承flags的列表
* parentsPflags：*flag.FlagSet，包含所有父命令的全局flags列表
* globNormFunc：func(f *flag.FlagSet, name string) flag.NormalizedName，全局标准化函数，该函数可用于该命令及其子命令的所有pflag集合
* usageFunc：func(*Command) error，用户自定义的用法函数
* usageTemplate：string，用户自定义的用法模板
* flagErrorFunc：func(*Command, error) error，当解析flags异常时返回错误
* helpTemplate：string，用户自定义的帮助文档模板
* helpFunc：func(*Command, []string)，用户自定义的帮助文档函数
* helpCommand：*Command，使用help命令时的Command对象，如果不指定则使用corba定义的默认帮助命令
* versionTemplate：string，用户自定义的版本模板
* inReader：io.Reader，用户定义的输入用于代替stdin
* outWriter：io.Writer，用户定义的输出用于代替stdout
* errWriter：io.Writer，用户定义的错误输出用于代替stderr

### Arguments（参数）
Args表示操作的对象或操作的条件，corba在Command中用args数组来保存所有入参，使用ValidArgs或ValidArgsFunction来进行参数校验。
另外，corba在Command中定义了Args来规范化、标准化入参。Args是一个类型为PersitionalArgs的函数，其标准定义为：func(cmd *Command, args []string) error.
corba定义了如下几个标准化函数：
* func legacyArgs(cmd *Command, args []string) error：该函数有三种处理方式：
    * 没有子命令的根命令可以接受任意参数
    * 带有子命令的根命令将执行子命令的有效性校验
    * 子命令接受任意参数
* func NoArgs(cmd *Command, args []string) error：如果输入参数则返回错误
* func OnlyValidArgs(cmd *Command, args []string) error：如果包含了不存在与ValidArgs中的参数则返回错误
* func ArbitraryArgs(cmd *Command, args []string) error：可输入任意参数
* func MinimumNArgs(n int) PositionalArgs：可输入的最少参数数量为n
* func MaximumNArgs(n int) PositionalArgs：可输入的最大参数数量为n
* func ExactArgs(n int) PositionalArgs：可输入的参数数量为n
* func ExactValidArgs(n int) PositionalArgs：ValidArgs中包含的参数数量为n
* func RangeArgs(min int, max int) PositionalArgs：输入的参数数量在{min, max}之间

### Flags（标志）
Flags是这些操作的修饰符，标志通过另一个库pflag进行解析和管理。pflag同样是由spf13提供的公共库。
其定义了Flag类型用于表示每一个标志：
* Name：string，标签的名称
* Shorthand：string，标签名称的字母缩写
* Usage：string，用法
* Value：Value，标签的值
* DefValue：string，标签的默认值，用于用法消息显示
* Changed：bool，如果用户设置了值（或使用默认值）时，设置为true
* NoOptDefVal：string，标签的默认值，如果该标志在命令行中没有任何选项
* Deprecated：string，如果该标志已废弃，则设置该字段用于提醒新的或现在用的标志
* Hidden：bool，允许设置为true的标志在帮助或用法中不显示
* ShorthandDeprecated：string，如果该标志的缩写被废弃，则使用该字段用于提醒新的或现在用的标志
* Annotations：map[string][]string，用于bash自动补全代码

Flag的Value是一个接口，其包含三个方法：
* String() string：Value值
* Set(string) error：设置Value值
* Type() string：Value值的类型

另外，pflag也定义了FlagSet来表示一组标志。

## 用法
创建Command
```go
var rootCmd = &cobra.Command{
  Use:   "demo",
  Short: "Demo is a very fast static site generator",
  Long: `A Fast and Flexible Static Site Generator built with
                love by spf13 and friends in Go.
                Complete documentation is available at http://hugo.spf13.com`,
  Run: func(cmd *cobra.Command, args []string) {
    // Do Stuff Here
  },
}

func Execute() {
  if err := rootCmd.Execute(); err != nil {
    fmt.Fprintln(os.Stderr, err)
    os.Exit(1)
  }
}
```
如果需要子命令，添加子命令
```go
func init() {
  rootCmd.AddCommand(versionCmd)
}

var versionCmd = &cobra.Command{
  Use:   "version",
  Short: "Print the version number of Hugo",
  Long:  `All software has versions. This is Hugo's`,
  Run: func(cmd *cobra.Command, args []string) {
    fmt.Println("Hugo Static Site Generator v0.9 -- HEAD")
  },
}
```
初始化命令参数和标志
```go
func init() {
	cobra.OnInitialize(initConfig)

	rootCmd.PersistentFlags().StringVar(&cfgFile, "config", "", "config file (default is $HOME/.cobra.yaml)")
	rootCmd.PersistentFlags().StringP("author", "a", "YOUR NAME", "author name for copyright attribution")
	rootCmd.PersistentFlags().StringVarP(&userLicense, "license", "l", "", "name of license for the project")
	rootCmd.PersistentFlags().Bool("viper", true, "use Viper for configuration")
	viper.BindPFlag("author", rootCmd.PersistentFlags().Lookup("author"))
	viper.BindPFlag("useViper", rootCmd.PersistentFlags().Lookup("viper"))
	viper.SetDefault("author", "NAME HERE <EMAIL ADDRESS>")
	viper.SetDefault("license", "apache")

	rootCmd.AddCommand(addCmd)
	rootCmd.AddCommand(initCmd)
}

func er(msg interface{}) {
	fmt.Println("Error:", msg)
	os.Exit(1)
}

func initConfig() {
	if cfgFile != "" {
		// Use config file from the flag.
		viper.SetConfigFile(cfgFile)
	} else {
		// Find home directory.
		home, err := homedir.Dir()
		if err != nil {
			er(err)
		}

		// Search config in home directory with name ".cobra" (without extension).
		viper.AddConfigPath(home)
		viper.SetConfigName(".cobra")
	}

	viper.AutomaticEnv()

	if err := viper.ReadInConfig(); err == nil {
		fmt.Println("Using config file:", viper.ConfigFileUsed())
	}
}
```
执行命令
```go
import (
  "{pathToYourApp}/cmd"
)

func main() {
  cmd.Execute()
}
```

## 命令执行流程
执行命令调用Command对象的Execute()方法，其执行步骤如下：
1. 调用ParseFlags()方法解析参数和标志
2. 执行preRun()方法进行一些初始化
3. 调用ValidateArgs()方法校验参数和标志
4. 如果定义了PersistentPreRun或PersistentPreRunE方法，则执行该方法；如果存在父节点，且父节点定义了PersistentPreRun或PersistentPreRunE方法，则继续执行父节点的方法
5. 如果定义了PreRun或PreRunE方法，则执行该方法
6. 指定Run或RunE方法
7. 如果定义了PostRun或PostRunE方法，则执行该方法
8. 如果定义了PersistentPostRun或PersistentPostRun方法，则执行该方法；如果存在父节点，且父节点定义了PersistentPostRun或PersistentPostRunE方法，则继续执行父节点的方法
