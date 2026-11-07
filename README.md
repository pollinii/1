# 1
Название: 1
Для кого: для всех 
Какая проблема: лень каждый раз искать или гуглить формулу это еще и долго онлайн-конвертеры перегружены рекламой
Что делает программа: принимает значение и единицы, мгновенно выдает результат
Функции: 
1. Конвертация длины (м, км, мили, футы, дюймы)
2. Конвертация веса (кг, г, фунты)
3. Конвертация температуры
4. Конвертация БЖУК
5. Решает математические уравнения 
6. Решает физические формулы 
Почему здесь можно использовать Go: легко расширять новыми единицами
package main
 
import (
    "bufio"
    "encoding/json"
    "fmt"
    "math"
    "os"
    "strconv"
    "strings"
    "time"
)
 
type Conversion struct {
    Category string  `json:"category"`
    Value    float64 `json:"value"`
    From     string  `json:"from"`
    To       string  `json:"to"`
    Result   float64 `json:"result"`
    Time     string  `json:"time"`
}
 
var lengthFactors = map[string]float64{
    "m":    1.0,
    "km":   1000.0,
    "mile": 1609.344,
    "ft":   0.3048,
    "in":   0.0254,
}
 
var massFactors = map[string]float64{
    "kg": 1.0,
    "g":  0.001,
    "lb": 0.45359237,
}
 
const historyFile = "history.json"
 
func main() {
    
    scanner := bufio.NewScanner(os.Stdin)
    var input string
    if scanner.Scan() {
        input = scanner.Text()
    }
 
    
    args := strings.Fields(input)
 
    
    if len(args) < 1 {
        showHelp()
        return
    }
 
    cmd := strings.ToLower(args[0])
 
    switch cmd {
    case "history":
        if err := showHistory(); err != nil {
            fmt.Fprintf(os.Stderr, "Ошибка чтения истории: %v\n", err)
            os.Exit(1)
        }
    case "length", "mass", "temperature":
        if len(args) < 4 {
            fmt.Fprintf(os.Stderr, "Ошибка: Недостаточно аргументов для конвертации.\nПример ввода в STDIN: %s 2 km m\n", cmd)
            os.Exit(1)
        }
        runConversion(cmd, args[1], args[2], args[3])
    case "solve":
        if len(args) < 3 {
            fmt.Fprintf(os.Stderr, "Ошибка: Недостаточно аргументов для решения уравнения.\nПример ввода в STDIN: solve 2 -4\n")
            os.Exit(1)
        }
        runEquation(args[1], args[2])
    default:
        showHelp()
    }
}
 
func showHelp() {
    fmt.Println("Утилита \"Конвертер величин с историей\"")
    fmt.Println("\nИнструкция для ввода в поле STDIN:")
    fmt.Println("  length <значение> <из> <в>      (Доступно: m, km, mile, ft, in)")
    fmt.Println("  mass <значение> <из> <в>        (Доступно: kg, g, lb)")
    fmt.Println("  temperature <значение> <из> <в> (Доступно: c, f, k)")
    fmt.Println("\nДля просмотра истории введите:")
    fmt.Println("  history")
    fmt.Println("\nДополнительно (решение ax + b = 0):")
    fmt.Println("  solve <a> <b>")
}
 
func runConversion(category, valStr, from, to string) {
    val, err := strconv.ParseFloat(valStr, 64)
    if err != nil {
        fmt.Fprintf(os.Stderr, "Ошибка: неверный формат числового значения '%s'\n", valStr)
        os.Exit(1)
    }
 
    
    if math.IsNaN(val) || math.IsInf(val, 0) {
        fmt.Fprintf(os.Stderr, "Ошибка: недопустимое числовое значение (NaN или Infinity)\n")
        os.Exit(1)
    }
 
    from = strings.ToLower(from)
    to = strings.ToLower(to)
 
    var result float64
    var convErr error
 
    switch category {
    case "length":
        if val < 0 {
            fmt.Fprintf(os.Stderr, "Ошибка: длина не может быть отрицательной\n")
            os.Exit(1)
        }
        result, convErr = convertByFactor(val, from, to, lengthFactors)
    case "mass":
        if val < 0 {
            fmt.Fprintf(os.Stderr, "Ошибка: масса не может быть отрицательной\n")
            os.Exit(1)
        }
        result, convErr = convertByFactor(val, from, to, massFactors)
    case "temperature":
        result, convErr = convertTemperature(val, from, to)
    }
 
    if convErr != nil {
        fmt.Fprintf(os.Stderr, "Ошибка конвертации: %v\n", convErr)
        os.Exit(1)
    }
 
    
    if math.IsNaN(result) || math.IsInf(result, 0) {
        fmt.Fprintf(os.Stderr, "Ошибка: результат вычислений некорректен (NaN или Infinity)\n")
        os.Exit(1)
    }
 
    
    fmt.Printf("%.4f %s = %.4f %s\n", val, from, result, to)
 
    
    op := Conversion{
        Category: category,
        Value:    val,
        From:     from,
        To:       to,
        Result:   result,
        Time:     time.Now().Format("2006-01-02 15:04:05"),
    }
    if err := saveToHistory(op); err != nil {
        fmt.Fprintf(os.Stderr, "Предупреждение по истории: %v\n", err)
    }
}
 
func convertByFactor(value float64, from, to string, factors map[string]float64) (float64, error) {
    fromFactor, ok := factors[from]
    if !ok {
        return 0, fmt.Errorf("неизвестная исходная единица: %s", from)
    }
    toFactor, ok := factors[to]
    if !ok {
        return 0, fmt.Errorf("неизвестная конечная единица: %s", to)
    }
    return value * fromFactor / toFactor, nil
}
 
func convertTemperature(value float64, from, to string) (float64, error) {
    if from != "c" && from != "f" && from != "k" {
        return 0, fmt.Errorf("неизвестная исходная единица температуры: %s", from)
    }
    if to != "c" && to != "f" && to != "k" {
        return 0, fmt.Errorf("неизвестная конечная единица температуры: %s", to)
    }
 
    switch from {
    case "c":
        if value < -273.15 {
            return 0, fmt.Errorf("температура ниже абсолютного нуля (-273.15 C)")
        }
    case "f":
        if value < -459.67 {
            return 0, fmt.Errorf("температура ниже абсолютного нуля (-459.67 F)")
        }
    case "k":
        if value < 0 {
            return 0, fmt.Errorf("температура ниже абсолютного нуля (0 K)")
        }
    }
 
    var cVal float64
    switch from {
    case "c":
        cVal = value
    case "f":
        cVal = (value - 32.0) * 5.0 / 9.0
    case "k":
        cVal = value - 273.15
    }
 
    var resVal float64
    switch to {
    case "c":
        resVal = cVal
    case "f":
        resVal = cVal*9.0/5.0 + 32.0
    case "k":
        resVal = cVal + 273.15
    }
 
    return resVal, nil
}
 
func loadHistory() ([]Conversion, error) {
    if _, err := os.Stat(historyFile); os.IsNotExist(err) {
        return []Conversion{}, nil
    }
 
    data, err := os.ReadFile(historyFile)
    if err != nil {
        return nil, fmt.Errorf("не удалось прочитать файл истории: %w", err)
    }
 
    var history []Conversion
    if err := json.Unmarshal(data, &history); err != nil {
        return nil, fmt.Errorf("файл истории поврежден: %w", err)
    }
 
    return history, nil
}
 
func saveToHistory(op Conversion) error {
    history, err := loadHistory()
    if err != nil {
        return err
    }
 
    history = append(history, op)
    if len(history) > 10 {
        history = history[len(history)-10:]
    }
 
    data, err := json.MarshalIndent(history, "", "  ")
    if err != nil {
        return fmt.Errorf("ошибка сериализации JSON: %w", err)
    }
 
    return os.WriteFile(historyFile, data, 0644)
}
 
func showHistory() error {
    history, err := loadHistory()
    if err != nil {
        return err
    }
 
    if len(history) == 0 {
        fmt.Println("История пуста.")
        return nil
    }
 
    fmt.Println("=== Последние 10 успешных операций ===")
    for i, op := range history {
        fmt.Printf("[%d] %s | %s: %.4f %s -> %.4f %s\n",
            i+1, op.Time, op.Category, op.Value, op.From, op.Result, op.To)
    }
    return nil
}
 
func runEquation(aStr, bStr string) {
    a, errA := strconv.ParseFloat(aStr, 64)
    b, errB := strconv.ParseFloat(bStr, 64)
 
    if errA != nil || errB != nil {
        fmt.Fprintf(os.Stderr, "Ошибка: неверный формат коэффициентов уравнения\n")
        os.Exit(1)
    }
 
    if math.IsNaN(a) || math.IsInf(a, 0) || math.IsNaN(b) || math.IsInf(b, 0) {
        fmt.Fprintf(os.Stderr, "Ошибка: коэффициенты не должны быть NaN или Infinity\n")
        os.Exit(1)
    }
 
    fmt.Printf("Решаем уравнение: (%.2f)x + (%.2f) = 0\n", a, b)
 
    if a == 0 {
        if b == 0 {
            fmt.Println("Результат: решений бесконечно много (0x + 0 = 0)")
        } else {
            fmt.Println("Результат: корней нет (равенство невыполнимо)")
        }
    } else {
        x := -b / a
        fmt.Printf("Результат: x = %.4f\n", x)
    }
}
