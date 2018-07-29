---
title: "RPN実装サンプル"
date: 2018-07-30T01:45:45+09:00
draft: false
tags: ["Kotlin"]
---
<script src="https://unpkg.com/kotlin-playground@1" data-selector="code"></script>

```kotlin
import java.util.*

enum class ItemType(val weight: Int? = null) {
    NUMBER(0),
    LEFT_BRA,
    RIGHT_BRA,
    PLUS(2),
    MINUS(2),
    MULTI(1),
    DIV(1)
}

data class Command(
        val type: ItemType,
        val text: String? = null
)

fun List<Command>.toRpn(): List<Command> {
    val returnList: ArrayList<Command> = ArrayList()
    val stack: Stack<Command> = Stack()

    this.forEach {
        when (it.type) {
            ItemType.RIGHT_BRA -> {
                val leftBraIndex =
                        stack.reversed().withIndex()
                                .firstOrNull {
                                    it.value.type == ItemType.LEFT_BRA
                                }?.index

                if (leftBraIndex != null) {
                    (0 until leftBraIndex).forEach {
                        returnList.add(stack.pop())
                    }
                    stack.pop()
                }
            }

            else -> {
                while (stack.isNotEmpty()) {
                    val latestStackWeight = stack.last().type.weight ?: break
                    val wight = it.type.weight ?: break

                    if (latestStackWeight > wight) break
                    else returnList.add(stack.pop())
                }

                stack.push(it)
            }
        }
    }

    returnList.addAll(stack.reversed())

    return returnList
}

fun main(args: Array<String>) {
    var formula = listOf(
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "13"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "4"),
        Command(ItemType.RIGHT_BRA, ")")
    )

    println ("original: ${formula.map { it.text }}")
    println ("RPN: ${formula.toRpn().map { it.text }}")
    println ("")

    formula = listOf(
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "3")
    )

    println ("original: ${formula.map { it.text }}")
    println ("RPN: ${formula.toRpn().map { it.text }}")
}
```
<br />
<br />
<br />
```kotlin
import java.util.*

enum class ItemType(val weight: Int? = null) {
    NUMBER(0),
    LEFT_BRA,
    RIGHT_BRA,
    PLUS(2),
    MINUS(2),
    MULTI(1),
    DIV(1)
}

data class Command(
        val type: ItemType,
        val text: String? = null
)

fun List<Command>.toRpn(): List<Command> {
    val returnList: ArrayList<Command> = ArrayList()
    val stack: Stack<Command> = Stack()

    this.forEach {
        when (it.type) {
            ItemType.RIGHT_BRA -> {
                val leftBraIndex =
                        stack.reversed().withIndex()
                                .firstOrNull {
                                    it.value.type == ItemType.LEFT_BRA
                                }?.index

                if (leftBraIndex != null) {
                    (0 until leftBraIndex).forEach {
                        returnList.add(stack.pop())
                    }
                    stack.pop()
                }
            }

            else -> {
                while (stack.isNotEmpty()) {
                    val latestStackWeight = stack.last().type.weight ?: break
                    val wight = it.type.weight ?: break

                    if (latestStackWeight > wight) break
                    else returnList.add(stack.pop())
                }

                stack.push(it)
            }
        }
    }

    returnList.addAll(stack.reversed())

    return returnList
}
//sampleStart
fun List<Command>.calculate(): Double {
        val stack: Stack<Double> = Stack()

        this.forEach {
            when (it.type) {
                ItemType.NUMBER -> {
                    stack.push(
                            it.text?.toDouble()
                                        ?: throw IllegalStateException()
                    )
                }

                ItemType.PLUS -> {
                    val a = stack.pop()
                    val b = stack.pop()
                    stack.push(b + a)
                }

                ItemType.MINUS -> {
                    val a = stack.pop()
                    val b = stack.pop()
                    stack.add(b - a)
                }

                ItemType.MULTI -> {
                    val a = stack.pop()
                    val b = stack.pop()
                    stack.add(b * a)
                }

                ItemType.DIV -> {
                    val a = stack.pop()
                    val b = stack.pop()
                    stack.add(b / a)
                }
            }
        }

        return stack.pop()
}

fun main(args: Array<String>) {
    var formula = listOf(
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "13"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "4"),
        Command(ItemType.RIGHT_BRA, ")")
    )

    println ("original: ${formula.map { it.text }}")
    println ("RPN: ${formula.toRpn().map { it.text }}")
    println ("calculate result: ${formula.toRpn().calculate()}")
    println ("")

    formula = listOf(
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "3")
    )

    println ("original: ${formula.map { it.text }}")
    println ("RPN: ${formula.toRpn().map { it.text }}")
    println ("calculate result: ${formula.toRpn().calculate()}")
}
//sampleEnd
```

From: [Flical](https://github.com/geckour/Flical)
