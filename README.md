# 🎛️ Modern C++ for JUCE Developers — Cheat Sheet
*A practical guide for audio plugin & DSP development (C++11 → C++26)*

---

##  1. Smart Pointers
### When to use
- **`std::unique_ptr<T>`** — owning Components, DSP objects  
- **`std::shared_ptr<T>`** — shared state (rare in audio threads)  
- **`std::weak_ptr<T>`** — break cycles  
- Prefer **`std::make_unique<T>`**  

### Example
```cpp
class MainComponent : public juce::Component {
public:
    MainComponent() {
        addAndMakeVisible(slider = std::make_unique<juce::Slider>());
    }

private:
    std::unique_ptr<juce::Slider> slider;
};
```

2. Move semantics (C++11)

Avoids unnecessary copies of buffers and DSP objects.

```cpp
juce::AudioBuffer<float> makeBuffer() {
    juce::AudioBuffer<float> buf(2, 512);
    return buf; // moved, not copied
}

auto buffer = makeBuffer();
```

3. Lambdas (C++11-20)

Great for DSP kernels, callbacks, and UI logic

```cpp
auto saturate = [](float x) { return std::tanh(x); };
auto scale    = [](auto x, auto g) { return x * g; }; // C++14
```

4. Modern concurrency
Use for background tasks, not audio thread work.
```cpp
auto result = std::async(std::launch::async, [] {
    return loadPresetFromDisk();
});

```

Coroutines (C++20)

```cpp
task loadPresetAsync() {
    co_await std::suspend_always{};
    // load preset
```
5. `optional`, `variant`, `expected`
```cpp
std::optional<float> getGain(bool ok) {
    if (!ok) return std::nullopt;
    return 0.8f;
}

using ParamValue = std::variant<float, int, bool>;

std::expected<float, juce::String> loadGain(int id) {
    if (id < 0) return std::unexpected("Invalid ID");
    return 0.75f;
}


```
6. `string_view` (C++17)
Zero‑allocation string handling.

```cpp
void parse(std::string_view msg) {
    if (msg.starts_with("note")) { /* ... */ } 
```

7. `constexpr`  DSP (C++11->C++20)
Compile‑time LUTs, waveshapers, filters.
```cpp
constexpr auto table = [] {
    std::array<float, 512> t{};
    for (size_t i = 0; i < t.size(); ++i)
        t[i] = std::sin(i * 0.01f);
    return t;
}();
```

8. Ranges (C++20 -> C++23)

 Cleaner buffer transforms.
```cpp
std::ranges::transform(buffer, buffer.begin(),
    [](float x) { return x * 0.5f; });
```

Chunking (C++23)

```cpp
for (auto chunk : buffer | std::views::chunk(64)) {
    // process 64-sample blocks
}

```

9. Concepts (C++20)
Type‑safe DSP templates.

```cpp
template <std::floating_point T>
T processSample(T x) { return x * 0.5; }

```
10. Modules (C++20-> C++26)

```cpp
export module dsp;
export float gain(float x) { return x * 0.5f; }

```

11. Pattern Matching & Reflection (C++26)

Useful for MIDI, parameter routing, UI events.

**Pattern matching (proposed)**

```cpp
match (event) {
    case NoteOn{n, v}: handleNoteOn(n, v);
    case NoteOff{n}:   handleNoteOff(n);
    default:           break;
}

```

### C++ Standards Summary Table 

| Standard | Must‑Know Additions |
| --- | --- |
| **C++98** | RAII, templates, STL |
| **C++03** | Rule of Three solidified |
| **C++11** | Move semantics, lambdas, auto, smart pointers |
| **C++14** | Generic lambdas, make_unique |
| **C++17** | optional, variant, string_view, structured bindings |
| **C++20** | concepts, ranges, coroutines, modules |
| **C++23** | expected, mdspan, print, more ranges |
| **C++26** | reflection, pattern matching, senders/receivers |

### JUCE pro tips 

Prefer value semantics for `AudioBuffer<float>`

Use `std::unique_ptr` for Components

Use `std::array` for fixed DSP buffers

Avoid `std::function` in audio thread (allocates)

Avoid virtual calls in tight DSP loops

Avoid allocations in audio thread

Use lambdas for parameter attachments

Use `dsp::ProcessorChain` with templates

### Anti‑Patterns to Avoid 

❌ Raw new / delete  
❌ Raw owning pointers
❌ std::vector::push_back in audio thread
❌ std::string in audio thread
❌ Exceptions in real‑time code
❌ std::function in DSP
❌ Virtual dispatch in sample loops

### MinimalModern DSP Processor Template

```cpp
class GainProcessor {
public:
    explicit GainProcessor(float g = 1.0f) : gain(g) {}

    void process(juce::AudioBuffer<float>& buffer) noexcept {
        auto* left  = buffer.getWritePointer(0);
        auto* right = buffer.getWritePointer(1);

        for (int i = 0; i < buffer.getNumSamples(); ++i) {
            left[i]  *= gain;
            right[i] *= gain;
        }
    }

    void setGain(float g) noexcept { gain = g; }

private:
    float gain{};
};

```
