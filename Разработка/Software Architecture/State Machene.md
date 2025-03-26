Вот реализация интерфейса, базового класса, Fluent API и пример использования конечного автомата для опросников на языке C#.

### Полный пример кода:

```csharp
using System;
using System.Collections.Generic;

namespace StateMachineExample
{
    // Интерфейс конечного автомата
    public interface IStateMachine<TState> where TState : Enum
    {
        TState CurrentState { get; }
        void AddTransition(TState source, TState destination, Action action);
        void MoveToState(TState destination);
    }

    // Базовый класс конечного автомата
    public class StateMachine<TState> : IStateMachine<TState> where TState : Enum
    {
        private readonly Dictionary<(TState Source, TState Destination), Action> _transitions = new();

        public TState CurrentState { get; private set; }

        public StateMachine(TState initialState)
        {
            CurrentState = initialState;
        }

        public void AddTransition(TState source, TState destination, Action action)
        {
            var key = (source, destination);
            if (!_transitions.ContainsKey(key))
            {
                _transitions[key] = action;
            }
            else
            {
                throw new InvalidOperationException($"Transition from {source} to {destination} already exists.");
            }
        }

        public void MoveToState(TState destination)
        {
            var key = (CurrentState, destination);
            if (_transitions.TryGetValue(key, out var action))
            {
                action?.Invoke();
                CurrentState = destination;
            }
            else
            {
                throw new InvalidOperationException($"Invalid transition from {CurrentState} to {destination}.");
            }
        }
    }

    // Расширения для Fluent API
    public static class StateMachineExtensions
    {
        public static StateMachine<TState> Configure<TState>(this StateMachine<TState> stateMachine, Action<StateMachineConfigurator<TState>> configure) where TState : Enum
        {
            var configurator = new StateMachineConfigurator<TState>(stateMachine);
            configure(configurator);
            return stateMachine;
        }
    }

    // Класс конфигуратора для Fluent API
    public class StateMachineConfigurator<TState> where TState : Enum
    {
        private readonly StateMachine<TState> _stateMachine;

        public StateMachineConfigurator(StateMachine<TState> stateMachine)
        {
            _stateMachine = stateMachine;
        }

        public StateMachineConfigurator<TState> AddTransition(TState source, TState destination, Action action)
        {
            _stateMachine.AddTransition(source, destination, action);
            return this;
        }
    }

    // Пример использования
    public enum SurveyState
    {
        Start,
        Question1,
        Question2,
        End
    }

    class Program
    {
        static void Main(string[] args)
        {
            // Создаем конечный автомат с начальным состоянием
            var surveyStateMachine = new StateMachine<SurveyState>(SurveyState.Start);

            // Конфигурируем конечный автомат
            surveyStateMachine.Configure(config =>
            {
                config.AddTransition(SurveyState.Start, SurveyState.Question1, () => Console.WriteLine("Moving to Question 1"));
                config.AddTransition(SurveyState.Question1, SurveyState.Question2, () => Console.WriteLine("Moving to Question 2"));
                config.AddTransition(SurveyState.Question2, SurveyState.End, () => Console.WriteLine("Survey complete!"));
            });

            // Используем конечный автомат
            try
            {
                Console.WriteLine($"Current State: {surveyStateMachine.CurrentState}");
                surveyStateMachine.MoveToState(SurveyState.Question1); // Переход к Question1
                Console.WriteLine($"Current State: {surveyStateMachine.CurrentState}");
                surveyStateMachine.MoveToState(SurveyState.Question2); // Переход к Question2
                Console.WriteLine($"Current State: {surveyStateMachine.CurrentState}");
                surveyStateMachine.MoveToState(SurveyState.End); // Переход к End
                Console.WriteLine($"Current State: {surveyStateMachine.CurrentState}");
            }
            catch (Exception ex)
            {
                Console.WriteLine($"Error: {ex.Message}");
            }
        }
    }
}
```

---

### Объяснение кода:

1. **Интерфейс `IStateMachine<TState>`**:
    
    - Обеспечивает типобезопасность при работе с состояниями.
    - Содержит базовые методы для добавления переходов и выполнения перехода между состояниями.
2. **Базовый класс `StateMachine<TState>`**:
    
    - Хранит текущее состояние и словарь переходов.
    - Реализует метод `MoveToState`, который проверяет доступность перехода и выполняет соответствующую логику.
3. **Fluent API**:
    
    - Реализовано через расширения и класс `StateMachineConfigurator<TState>`.
    - Позволяет удобно добавлять переходы через цепочку вызовов.
4. **Пример использования**:
    
    - Определены состояния опросника: `Start`, `Question1`, `Question2`, `End`.
    - Создан и настроен конечный автомат.
    - Демонстрация переходов между состояниями с выполнением соответствующей логики.

---

С этим кодом ты можешь легко создавать и конфигурировать конечные автоматы для реализации различных опросников или других сценариев! 😊
