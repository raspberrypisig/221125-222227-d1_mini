<script>
    import { post_www_url_encoded } from "./sendcommmand.js";
    import { drawHangmanPart } from "./drawhangman.js";

    let categories = [
        {
            id: "country",
            name: "Country",
            url: "https://cdn.jsdelivr.net/gh/raspberrypisig/corpora/custom/countries-limited.json",
        },
        {
            id: "fruit",
            name: "Fruit",
            url: "https://cdn.jsdelivr.net/gh/raspberrypisig/corpora/custom/fruits-limited.json",
        },
        {
            id: "vegetable",
            name: "Vegetable",
            url: "https://cdn.jsdelivr.net/gh/raspberrypisig/corpora/custom/vegetables-limited.json",
        },
        {
            id: "animal",
            name: "Animal",
            url: "https://cdn.jsdelivr.net/gh/raspberrypisig/corpora/custom/animals-limited.json",
        },
    ];
    const HANGMAN_GAMESTATE = {
        STARTGAME: 0,
        SELECTCATEGORY: 1,
        PLAYING: 2,
        ENDGAME: 3,
    };
    let hangmanGameState = HANGMAN_GAMESTATE.SELECTCATEGORY;
    let answer;
    let revealedLetters;
    let categorySelection = "";
    let itemCapitalized;
    let incorrectAttempts = 0;
    let incorrrectLetters = "";
    let selectedCategoryName;
    const availableLettersCopy = [
        "A",
        "B",
        "C",
        "D",
        "E",
        "F",
        "G",
        "H",
        "I",
        "J",
        "K",
        "L",
        "M",
        "N",
        "O",
        "P",
        "Q",
        "R",
        "S",
        "T",
        "U",
        "V",
        "W",
        "X",
        "Y",
        "Z",
    ];
    let availableLetters = availableLettersCopy;

    function startNewGame() {
        hangmanGameState = HANGMAN_GAMESTATE.SELECTCATEGORY;
        revealedLetters = "";
        incorrectAttempts = 0;
        incorrrectLetters = "";
        availableLetters = availableLettersCopy;
    }

    function displayIncorrectLetters(letters) {
        const l = letters.split("").reduce(function (arr, v, i) {
            return arr.concat([v, " "]);
        }, []);
        if (letters.length >= 5) {
            post_www_url_encoded({
                command: "R",
                x1: 48,
                y1: 0,
                text1: l.slice(0, 8).join(""),
                fontIndex: 1,
            });

            post_www_url_encoded({
                command: "R",
                x1: 48,
                y1: 8,
                text1: l.slice(8).join(""),
                fontIndex: 1,
            });
        } else {
            console.log(l.join(""));
            post_www_url_encoded({
                command: "R",
                x1: 48,
                y1: 0,
                text1: l.join(""),
                fontIndex: 1,
            });
        }
    }

    function selectCategory(id) {
        const selectedCategories = categories.filter(
            (category) => category.id === id
        );
        const selectedCategory = selectedCategories[0];
        categorySelection = selectedCategory.name;
        selectedCategoryName = selectedCategory.name.toLocaleUpperCase();
        const selectedCategoryURL = selectedCategory.url;

        hangmanGameState = HANGMAN_GAMESTATE.PLAYING;

        // converted github link to jsdelivr CDN
        fetch(selectedCategoryURL)
            .then((response) => response.json())
            .then((data) => {
                console.log(data);
                const item = data[Math.floor(Math.random() * data.length)];
                itemCapitalized = item.toUpperCase();

                const categoryNameSize = selectedCategoryName.length;
                const itemSize = itemCapitalized.length;
                answer = itemCapitalized;
                console.log("category", selectedCategoryName);
                console.log("category name size:", categoryNameSize);
                console.log("random item:", itemCapitalized);
                console.log("random item size:", itemSize);
                post_www_url_encoded({
                    command: "@",
                });

                const spaces = Array(itemSize).fill(" ");
                const underscores = Array(itemSize).fill("_");
                revealedLetters = underscores;

                const words = underscores.reduce(function (arr, v, i) {
                    return arr.concat([v, " "]);
                }, []);

                console.log(words);

                drawHangmanPart(incorrectAttempts);

                post_www_url_encoded({
                    command: "M",
                    x1: 0,
                    y1: 16,
                    text1: selectedCategoryName,
                    x2: 0,
                    y2: 24,
                    text: words.join(""),
                    fontIndex: 1,
                });
            });
    }

    function guessLetter(letter) {
        availableLetters = availableLetters.filter((elem) => elem !== letter);
        let text = "";
        if (itemCapitalized.indexOf(letter) !== -1) {
            console.log("letter in word");
            console.log(revealedLetters);
            console.log(answer);
            const indexes = answer
                .split("")
                .map((v, i) => (v === letter ? i : -1))
                .filter((elem) => elem !== -1);
            console.log(indexes);
            for (let i = 0; i < indexes.length; i++) {
                const j = indexes[i];
                revealedLetters[j] = letter;
            }
            console.log(revealedLetters);
            const b = revealedLetters.reduce(function (arr, v, i) {
                return arr.concat([v, " "]);
            }, []);
            text = b.join("");
            console.log(text);
            const trimmedtext = text.replace(/\s+/g, "");
            console.log(trimmedtext);
            if (trimmedtext === answer) {
                console.log("word is revealed.");
                hangmanGameState = HANGMAN_GAMESTATE.ENDGAME;
            }
            post_www_url_encoded({
                command: "M",
                x1: 0,
                y1: 16,
                text1: selectedCategoryName,
                x2: 0,
                y2: 24,
                text: text,
                fontIndex: 1,
            });
        } else {
            console.log("letter not in word.");
            incorrectAttempts = incorrectAttempts + 1;
            incorrrectLetters = incorrrectLetters + letter;
            if (incorrectAttempts === 7) {
                hangmanGameState = HANGMAN_GAMESTATE.ENDGAME;
                const formattedAnswer = answer
                    .split("")
                    .reduce(function (arr, v, i) {
                        return arr.concat([v, " "]);
                    }, []);
                console.log(formattedAnswer.join(""));
                post_www_url_encoded({
                    command: "M",
                    x1: 0,
                    y1: 16,
                    text1: selectedCategoryName,
                    x2: 0,
                    y2: 24,
                    text: formattedAnswer.join(""),
                    fontIndex: 1,
                });
            }

            displayIncorrectLetters(incorrrectLetters);
            drawHangmanPart(incorrectAttempts);
        }
        console.log(incorrectAttempts);
    }
</script>

<main>
    <h1>Hangman</h1>
    {#if hangmanGameState == HANGMAN_GAMESTATE.ENDGAME}
        <section>
            <button on:click={() => startNewGame()}>NEW GAME</button>
            <div>ANSWER: <span>{answer}</span></div>
        </section>
    {/if}
    {#if hangmanGameState == HANGMAN_GAMESTATE.SELECTCATEGORY}
        <section>
            <h2>Select category</h2>
            <ul>
                {#each categories as category}
                    <li>
                        <a
                            class="categories"
                            id={category.id}
                            on:click={() => selectCategory(category.id)}
                            >{category.name}</a
                        >
                    </li>
                {/each}
            </ul>
        </section>
    {/if}
    {#if hangmanGameState == HANGMAN_GAMESTATE.PLAYING}
        <section>
            <h2>Category</h2>
            <p>{categorySelection}</p>
        </section>
    {/if}
    {#if hangmanGameState == HANGMAN_GAMESTATE.PLAYING}
        <section>
            <h2>Select Letter</h2>
            {#each availableLetters as letter}
                <button class="letters" on:click={() => guessLetter(letter)}
                    >{letter}</button
                >
            {/each}
        </section>
    {/if}
</main>

<style lang="scss">
    .letters {
        margin-left: 30px;
    }
</style>
