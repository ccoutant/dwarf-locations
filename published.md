The core idea behind making a class member **`published`** in Object
Pascal (Borland Delphi and Free Pascal) is to enable **Rapid
Application Development (RAD)** by directing the compiler to generate
**Run-Time Type Information (RTTI)** for that member.

While standard compiled languages like C++ historically discarded
variable and method names after compilation, `published` bridges the
gap between a statically typed, natively compiled language and a
dynamic visual development environment.

---

### Historical Context: The Delphi Revolution (1995)

In the early 1990s, **Borland Pascal 7.0** featured standard Object
Pascal with basic object-oriented constructs (`public`, `private`, and
`protected`). However, Microsoft Visual Basic was dominating desktop
GUI development because of its visual "drag-and-drop" editor and
property inspector.

To compete, Borland (led by Anders Hejlsberg) designed **Delphi
1.0**. Borland wanted the raw speed, type safety, and direct machine
code generation of Pascal, but combined with the dynamic component
manipulation of Visual Basic.

To achieve this, the language needed a way for an IDE (and runtime
libraries) to ask an arbitrary object at runtime:

1. *What properties do you have?*
2. *What are their current values?*
3. *What event handlers (method pointers) can I hook into you?*

The solution was the **`published`** keyword and compiler-generated
RTTI.

---

### Language Visibility vs. Compiler Behavior

Understanding `published` requires separating its visibility semantics
from its compiler emission:

| Aspect | `public` | `published` |
| --- | --- | --- |
| **Accessibility** | Usable anywhere in the program. | Usable anywhere in the program (identical to `public`). |
| **Metadata (RTTI)** | Compiler strips field, method, and property names from the final binary (in legacy RTTI). | Compiler embeds metadata tables containing property names, types, and memory offsets directly into the executable binary. |

---

### The Three Core Applications of `published`

```pascal
type
  TMyButton = class(TComponent)
  private
    FCaption: string;
    FOnClick: TNotifyEvent;
  published
    property Caption: string read FCaption write FCaption;
    property OnClick: TNotifyEvent read FOnClick write FOnClick;
  end;

```

#### 1. The Design-Time Object Inspector

When you drop a component onto a form in Delphi or Lazarus (Free
Pascal IDE), the IDE reflects upon the instance using its RTTI
tables. It iterates over all `published` properties and automatically
populates the **Object Inspector** window, allowing developers to
visually edit string values, colors, dropdowns, and flags without
writing code.

#### 2. Component Streaming and Form Serialization (`.dfm` / `.lfm`)

When you save a visual layout in Delphi, it writes out a form file
(`.dfm` in Delphi, `.lfm` in Lazarus) containing component properties.

* At runtime, the streaming engine (`TReader` and `TWriter` in
  `Classes.pas`) opens the form file, reads string keys like `"Caption
  = 'Submit'"`, and uses the class's `published` RTTI to find where
  `Caption` lives in memory and assign the value automatically.

#### 3. Dynamic Event Wiring (Method Pointers)

Event handlers (such as `OnClick`) are stored as method pointers. When
you double-click a button in the IDE, Delphi inserts a method
declaration like `procedure Button1Click(Sender: TObject)` into your
form class.

* Because this handler is in the form's `published` section, the form
  streaming engine looks up the method name `"Button1Click"` in the
  RTTI tables at runtime and binds the button's `OnClick` pointer
  directly to that procedure.

---

### Rules and Technical Nuances

* **The `{$M+}` / `TPersistent` Requirement:** The compiler will only
generate RTTI for `published` sections if the class is compiled with
the `{$M+}` (`{$TYPEINFO ON}`) compiler directive, or if it inherits
from a class that already has `{$M+}` enabled. In practice, almost all
published components inherit from `TPersistent` or `TComponent`.

* **Implicit Default Section in Forms:** In Delphi and Lazarus form
declarations, any member declared before the first explicit access
specifier (`private`, `public`, etc.) defaults to **`published`**:

```pascal
type
  TForm1 = class(TForm)
    Button1: TButton;      // Automatically 'published'
    procedure Button1Click(Sender: TObject); // Automatically 'published'
  private
    FSecret: Integer;      // Explicitly private
  end;

```


* **Type Restrictions (Legacy RTTI):**

Historically, you could not publish *any* arbitrary data type. A
`published` property had to be a type supported by the streaming
engine—such as integers, strings, floating-point numbers, enums, sets,
class references, interfaces, or method pointers. Pointer types,
multidimensional raw arrays, and unmanaged records could not be
`published`.

---

### Evolution in Modern Pascal

* **Free Pascal / Lazarus:** Free Pascal maintains 100% compatibility
    with traditional Object Pascal `published` semantics for its
    Component Library (LCL).

* **Delphi 2010 Extended RTTI:** Starting in Delphi 2010, Embarcadero
    introduced a modern reflection framework (`{$RTTI}` directive and
    `System.RTTI` unit). This allows full reflection over `public`,
    `protected`, and `private` members similar to C# or Java. However,
    `published` remains the foundational mechanism for IDE component
    design and lightweight form streaming.