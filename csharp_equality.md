# Как работать с равенством в С#

Реализаия проверки равенства между объектами класса в любом популярном ООП языке программирования является не тривиальной задачей. 

Например, в С# для этого используются следующие методы, встроенные в каждый объект:
- object.Equals
- оператор == 
- ReferenceEquals

Равенство по значению это процесс определения равенства двух объектов путем сравнения их содержания.

Такая задача является нетривиальной, так как мы можем столкнуться с разными типами объектов в зависимости от того, хранятся значения в Stack (значимый тип) или в Heap (ссылочный тип). 

При сравнении ссылочных типов сравниваются ссылки на область памяти. Если объекты имеют одинаковые внутренние значения, но при этом хранятся в разных областях памяти, то получим False при сравнении этих объектов. 

Как же эффективно реализовать сравнения типов данных по их внутреннему состоянию?

## Подключить IEquatable<T> классу 

Добавлю сразу: T - это класс.

Интерфейс `IEquatable` просто добавит следующий метод к необходимому классу:

Ничего сложного.

Интересно то, что при вызове `object.Equals(object o)`, мы получим вызов переопределенного метода `Equals` при сравнении объектов.

Объявим класс: 

```csharp
public class MyClass : IEquatable<Foo>{
	public int Num {get; init set;}
	public string Str {get; init set;}
	public DateTime Date {get; init set;}

	#region Equality

	public bool Equals(MyClass other){
		throw new NotImplementedException();
	}
			
	#endregion
}
```

## Применить метод Equals(T other)

Необходимо определить равны ли значения каждого отдельного поля. 

```csharp
public bool Equals(MyClass other){
	if(other == null) return false;
	return Num == other.Num &&
			Date == other.Date &&
			string.Equals(Str, other.Str);
}
```

В этом случае мы сравниваем все свойства, потому что они простые. Нам не хотелось бы столкнуться c ошибкой NullReferenceException, поэтому можно проверить является ли объект other == null.  Строка может иметь значение null, так что string.Equals вернет либо true, либо false без создания исключения, даже если одна или обе строки имеют значение null.

Что делать если сравниваем класс с его наследником? Например, класс MyExtendedClass наследует MyClass. Как тогда достигнуть равенство через проверку значения для MyClass?

## object.Equals(object o)

Дальше идет довольно-таки шаблонная код:

Переопределим метод object.Equals, заменив его шаблонным кодом, который был построен на методе IEquatable<Foo>.Equals(Foo other).

```csharp
public class MyClass : IEquatable<MyClass>{
	public int Num {get; init set;}
	public string Str {get; init set;}
	public DateTime Date {get; init set;}

	#region Equality

	public bool Equals(MyClass other){
		if(other == null) return false;
		return Num == other.Num &&
			Date == other.Date &&
			string.Equals(Str, other.Str);
	}

	public override bool Equals(object obj){
		if (ReferenceEquals(null, obj)) return false;
        if (ReferenceEquals(this, obj)) return true;
        if (obj.GetType() != GetType()) return false;
		return Equals(obj as MyClass);
	}
			
	#endregion

```

1. Определить является ли obj == null, код вернет false;
2. Если же obj равен значению, код вернет true;
3. Проверить не равен ли тип obj текущему типу: если нет, то вернется false;
4. obj необходимо привести к MyClass и перенаправить на метод Equals(MyClass other)

## Уникальный GetHashCode

Итак, у нас есть возможность определить, равны ли два объекта MyClass по значению, но мы еще не переопределили функцию GetHashCode. Что это значит?

Хэш-код используется встроенными компонентами .NET Framework, поэтому очень важно переопределить GetHashCode.

Не важно, какое значение имеет хэш-код - нам важно только то, что это уникальный значение, которое может содержать только один объект с точно такими же значениями.

Мы будем использовать технику вычислений, которая обеспечивает воспроизводимый хэш-код для всех одинаковых по значению экземпляров MyClass и делает вероятность повторения хэшей крайне низкой.

```csharp
public class MyClass : IEquatable<MyClass>{
	public int Num {get; init set;}
	public string Str {get; init set;}
	public DateTime Date {get; init set;}

	#region Equality

	public bool Equals(MyClass other){
		if(other == null) return false;
		return Num == other.Num &&
			Date == other.Date &&
			string.Equals(Str, other.Str);
	}

	public override bool Equals(object obj){
		if (ReferenceEquals(null, obj)) return false;
        if (ReferenceEquals(this, obj)) return true;
        if (obj.GetType() != GetType()) return false;
		return Equals(obj as MyClass);
	}

    public override int GetHashCode(){
		unchecked{
			var hashCode = 13;
                hashCode = (hashCode * 397) ^ Num;
                hashCode = (hashCode * 397) ^ (!string.IsNullOrEmpty(Str) ? Str.GetHashCode() : 0) ;
                hashCode = (hashCode * 397) ^ Date.GetHashCode();
                return hashCode;
		}
	}			
			
	#endregion

```

Во-первых, необходимо использовать ключевое слово `unchecked`, чтобы сообщить CLR, что нам неважно, переполняется ли целочисленное значение или нет. Нам важно только значение.

Во-вторых, мы выбираем два разных простых числа, одно из которых будет использоваться как средство рандомизации хэша, а другое - в качестве хэш-множителя. Для каждого свойства, которое нам нужно включить в хэш-функцию, мы умножаем текущий хэш на простое число, а затем включаем хэш-код каждого отдельного поля.

Лично я использую побитовое умножение или (^), но вы можете использовать и сложение.

## В каких случаях необходимо перезаписывать GetHashCode?

Переопределять `GetHashCode` можно только в том случае, если вы работаете с неизменяемыми объектами.

Если свойства класса `MyClass` могут меняться и изменять результат `GetHashCode`, то любая коллекция (`List<MyClass>` и т.д.) будет вести себя непредсказуемо. Возможно будет выдавать исключение если хэш-код отдельного примера Foo в коллекции изменится.

