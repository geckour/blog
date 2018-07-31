---
title: "RPN実装サンプル"
date: 2018-07-30T01:45:45+09:00
draft: false
tags: ["Kotlin"]
---
<script src="https://unpkg.com/kotlin-playground@1" data-selector="code"></script>

# To RPN
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
    val stack: Deque<Command> = ArrayDeque()

    this.forEach {
        when (it.type) {
            ItemType.RIGHT_BRA -> {
                val leftBraIndex =
                        stack.withIndex()
                                .firstOrNull {
                                    it.value.type == ItemType.LEFT_BRA
                                }?.index

                if (leftBraIndex != null) {
                    (0 until leftBraIndex).forEach {
                        returnList.add(stack.removeFirst())
                    }
                    stack.removeFirst()
                }
            }

            else -> {
                while (stack.isNotEmpty()) {
                    val latestStackWeight = stack.first().type.weight ?: break
                    val wight = it.type.weight ?: break

                    if (latestStackWeight > wight) break
                    else returnList.add(stack.removeFirst())
                }

                stack.addFirst(it)
            }
        }
    }

    returnList.addAll(stack)

    return returnList
}

fun main(args: Array<String>) {
    //sampleStart
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

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    println ("")

    formula = listOf(
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "3")
    )

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    println ("")

    formula = listOf(
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "1"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "3"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "4"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "6"),
        Command(ItemType.RIGHT_BRA, ")")
    )

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    //sampleEnd
}
```

参考: [数式を逆ポーランド記法に変換 (Qiita)](https://qiita.com/ken-maki/items/02d89de02925e9fa12e5)

# Evaluate RPN
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
    val stack: Deque<Command> = ArrayDeque()

    this.forEach {
        when (it.type) {
            ItemType.RIGHT_BRA -> {
                val leftBraIndex =
                        stack.withIndex()
                                .firstOrNull {
                                    it.value.type == ItemType.LEFT_BRA
                                }?.index

                if (leftBraIndex != null) {
                    (0 until leftBraIndex).forEach {
                        returnList.add(stack.removeFirst())
                    }
                    stack.removeFirst()
                }
            }

            else -> {
                while (stack.isNotEmpty()) {
                    val latestStackWeight = stack.first().type.weight ?: break
                    val wight = it.type.weight ?: break

                    if (latestStackWeight > wight) break
                    else returnList.add(stack.removeFirst())
                }

                stack.addFirst(it)
            }
        }
    }

    returnList.addAll(stack)

    return returnList
}
//sampleStart
fun List<Command>.calculate(): Double {
        val stack: Deque<Double> = ArrayDeque()

        this.forEach {
            when (it.type) {
                ItemType.NUMBER -> {
                    stack.addFirst(
                            it.text?.toDouble()
                                        ?: throw IllegalStateException()
                    )
                }

                ItemType.PLUS -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b + a)
                }

                ItemType.MINUS -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b - a)
                }

                ItemType.MULTI -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b * a)
                }

                ItemType.DIV -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b / a)
                }
            }
        }

        return stack.removeFirst()
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

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    println ("calculate result: ${formula.toRpn().calculate()}")
    println ("")

    formula = listOf(
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "3")
    )

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    println ("calculate result: ${formula.toRpn().calculate()}")
    println ("")

    formula = listOf(
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "1"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "3"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "4"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "6"),
        Command(ItemType.RIGHT_BRA, ")")
    )

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    println ("calculate result: ${formula.toRpn().calculate()}")
}
//sampleEnd
```

# More complex
```kotlin
import kotlin.math.*
import java.util.*

enum class ItemType(val weight: Int? = null) {
    NUMBER(0),
    PI(0),
    E(0),
    LEFT_BRA,
    RIGHT_BRA,
    PLUS(4),
    MINUS(4),
    MULTI(3),
    DIV(3),
    POW(2),
    MOD(2),
    SQRT(1),
    SIN(1),
    COS(1),
    TAN(1),
    A_SIN(1),
    A_COS(1),
    A_TAN(1),
    LN(1),
    LOG10(1),
    LOG2(1),
    ABS(1)
}

data class Command(
        val type: ItemType,
        val text: String? = null
)

fun List<Command>.toRpn(): List<Command> {
    val returnList: ArrayList<Command> = ArrayList()
    val stack: Deque<Command> = ArrayDeque()

    this.forEach {
        when (it.type) {
            ItemType.RIGHT_BRA -> {
                val leftBraIndex =
                        stack.withIndex()
                                .firstOrNull {
                                    it.value.type == ItemType.LEFT_BRA
                                }?.index

                if (leftBraIndex != null) {
                    (0 until leftBraIndex).forEach {
                        returnList.add(stack.removeFirst())
                    }
                    stack.removeFirst()
                }
            }

            else -> {
                while (stack.isNotEmpty()) {
                    val latestStackWeight = stack.first().type.weight ?: break
                    val wight = it.type.weight ?: break

                    if (latestStackWeight > wight) break
                    else returnList.add(stack.removeFirst())
                }

                stack.addFirst(it)
            }
        }
    }

    returnList.addAll(stack)

    return returnList
}

fun List<Command>.calculate(): Double {
        val stack: Deque<Double> = ArrayDeque()

        this.forEach {
            when (it.type) {
                ItemType.NUMBER -> {
                    stack.addFirst(
                        it.text?.toDouble()
                        ?: throw IllegalStateException()
                    )
                }

                ItemType.PI -> {
                    stack.addFirst(PI)
                }

                ItemType.E -> {
                    stack.addFirst(E)
                }

                ItemType.PLUS -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b + a)
                }

                ItemType.MINUS -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b - a)
                }

                ItemType.MULTI -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b * a)
                }

                ItemType.DIV -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    val preResult = b.toDouble() / a.toDouble()
                    stack.addFirst(b / a)
                }

                ItemType.POW -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b.pow(a))
                }

                ItemType.MOD -> {
                    val a = stack.removeFirst()
                    val b = stack.removeFirst()
                    stack.addFirst(b % a)
                }

                ItemType.SQRT -> {
                    val a = stack.removeFirst()
                    stack.addFirst(sqrt(a))
                }

                ItemType.COS -> {
                    val a = stack.removeFirst()
                    stack.addFirst(cos(a))
                }

                ItemType.SIN -> {
                    val a = stack.removeFirst()
                    stack.addFirst(sin(a))
                }

                ItemType.TAN -> {
                    val a = stack.removeFirst()
                    stack.addFirst(tan(a))
                }

                ItemType.A_COS -> {
                    val a = stack.removeFirst()
                    stack.addFirst(acos(a))
                }

                ItemType.A_SIN -> {
                    val a = stack.removeFirst()
                    stack.addFirst(asin(a))
                }

                ItemType.A_TAN -> {
                    val a = stack.removeFirst()
                    stack.addFirst(atan(a))
                }

                ItemType.LN -> {
                    val a = stack.removeFirst()
                    stack.addFirst(ln(a))
                }

                ItemType.LOG10 -> {
                    val a = stack.removeFirst()
                    stack.addFirst(log10(a))
                }

                ItemType.LOG2 -> {
                    val a = stack.removeFirst()
                    stack.addFirst(log2(a))
                }

                ItemType.ABS -> {
                    val a = stack.removeFirst()
                    stack.addFirst(abs(a))
                }
            }
        }

        return stack.removeFirst()
}

fun main(args: Array<String>) {
//sampleStart
    var formula = listOf(
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.SQRT, "√"),
        Command(ItemType.NUMBER, "9"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "5"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.NUMBER, "13"),
        Command(ItemType.MOD, "%"),
        Command(ItemType.NUMBER, "3"),
        Command(ItemType.PLUS, "+"),
        Command(ItemType.NUMBER, "4"),
        Command(ItemType.RIGHT_BRA, ")"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.SIN, "sin"),
        Command(ItemType.LEFT_BRA, "("),
        Command(ItemType.PI, "π"),
        Command(ItemType.MULTI, "*"),
        Command(ItemType.NUMBER, "3"),
        Command(ItemType.DIV, "/"),
        Command(ItemType.NUMBER, "2"),
        Command(ItemType.RIGHT_BRA, ")")
    )

    println ("original: ${formula.map { it.text }.joinToString(" ")}")
    println ("RPN: ${formula.toRpn().map { it.text }.joinToString(" ")}")
    println ("calculate result: ${formula.toRpn().calculate()}")
    //sampleEnd
}
```

From: [Flical](https://github.com/geckour/Flical)
