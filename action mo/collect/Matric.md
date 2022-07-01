





```mermaid
classDiagram

	父类 <|-- 子类

	class Animal
	Animal: Name String
	Animal: Eat()
	Animal: Drink()
	
	class Tiger
	Animal <|-- Tiger
	Tiger: Name String
```

