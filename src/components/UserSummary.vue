<template>
    <section class="summary">
        <div class="container">

            <h2>Overview for <a target="_blank" :href="userLink">/u/{{username}}</a></h2>
            <p class="subtext">Joined reddit <strong>{{ date('relative', about.created_utc) }}</strong> ({{ date('utc', about.created_utc) }})</p>
            <p><small>*Data is available from 1000 comments and 1000 submissions ago (Reddit API limitations)</small></p>

            <hr>

            <summary-circles :totalComments="totalComments"
                             :totalSubmitted="totalSubmitted"
                             :controversiality="controversialityScore"
                             :karmaPerComment="karmaPerComment"
                             :karmaPerSubmitted="karmaPerSubmitted">
            </summary-circles>

            <hr>

            <h3>English-based text analysis</h3>
            <div class="row">
                <div class="col-md-6">
                    <kindness-meter :value="kindnessScore"></kindness-meter>
                </div>
                <div class="col-md-6">
                    <div>
                        <div class="center">
                            <img class="section-illustration" src="../assets/textcomplexity.png">
                        </div>
                        <h3>Text complexity</h3>
                        <p>Calculated with the Flesch Formula</p>
                        <p class="gradient-uppercase">{{textComplexity}}</p>
                    </div>
                </div>
            </div>

            <hr>

            <h3>Top subreddits</h3>
            <div class="input-group input-group-lg input-rangeslider">
                <input
                    type="range"
                    min="1"
                    max="100"
                    step="1"
                    v-model="numSubreddits"
                >
            </div>
            <ul class="d-flex flex-wrap justify-content-center">
                <li class="card card--dark" v-for="subreddit in topSubreddits">
                    <a target="_blank" :href="subredditLink(subreddit.name)">
                    /r/{{subreddit.name}}
                    <div>
                        <small>{{ subreddit.count }} {{ subreddit.count == 1 ? 'post' : 'posts' }} ({{ percentageOf(subreddit.count) }}%)</small>
                    </div>
                    </a>
                </li>
            </ul>

            <hr>

            <h3>Most frequently used words</h3>
            <div class="input-group input-group-lg input-rangeslider">
                <input
                    type="range"
                    min="1"
                    max="100"
                    step="1"
                    v-model="numFrequentWords"
                >
            </div>
            <ul class="d-flex flex-wrap justify-content-center">
                <li class="card card--dark" v-for="word in mostFrequentlyUsedWords">
                    <a target="_blank" :href="googleLink(word.name)">
                    {{ word.name }} <small>{{ word.frequency }} times</small>
                    </a>
                </li>
            </ul>

            <hr>

            <div class="row">
                <div class="col-md-6">
                    <graph-section id="commentActivity-chart"
                                   type="commentActivity"
                                   :days="days"
                                   :allDays="allDays"
                                   title="Comment activity over time">
                    </graph-section>
                </div>
                <div class="col-md-6">
                    <graph-section id="commentKarma-chart"
                                   type="commentKarma"
                                   :days="days"
                                   :allDays="allDays"
                                   title="Comment karma over time">
                    </graph-section>
                </div>
            </div>

            <hr>

            <div class="row">
                <div class="col-md-6">
                    <graph-section id="submittedActivity-chart"
                                   type="submittedActivity"
                                   :days="days"
                                   :allDays="allDays"
                                   title="Submission activity over time">
                    </graph-section>
                </div>
                <div class="col-md-6">
                    <graph-section id="submittedKarma-chart"
                                   type="submittedKarma"
                                   :days="days"
                                   :allDays="allDays"
                                   title="Submission karma over time">
                    </graph-section>
                </div>
            </div>

            <hr>

            <graph-section id="controversiality-chart"
                           type="controversiality"
                           :days="days"
                           :allDays="allDays"
                           title="Controversiality over time">
            </graph-section>

            <hr>

            <div class="row">
                <div class="col-md-6">
                    <h3>Best comment</h3>
                    <comment :data="mostUpvotedComment"><comment>
                </div>
                <div class="col-md-6">
                    <h3>Worst comment</h3>
                    <comment :data="mostDownvotedComment"><comment>
                </div>
            </div>

        </div>
    </section>
</template>

<script>
import SummaryCircles from 'components/SummaryCircles'
import KindnessMeter from 'components/KindnessMeter'
import GraphSection from 'components/GraphSection'
import Comment from 'components/Comment'

import moment from 'moment'

export default {
    name: 'user-summary',
    components: {
        SummaryCircles,
        KindnessMeter,
        GraphSection,
        Comment
    },
    props: ['username', 'about', 'comments', 'submitted', 'isLoading'],
    data() {
        return {
            subreddits: {},
            commentKarma: 0,
            submittedKarma: 0,
            subredditCounts: [],
            wordCounts: [],
            days: [],
            allDays: [],
            numSubreddits: 10,
            numFrequentWords: 15,
            badWordIncidence: 0,
            niceWordIncidence: 0,
            readingLevel: 0,
            commentsSortedByScore: null
        }
    },
    computed: {
        topSubreddits() {
            if (this.isLoading) { this.newUser = true; }

            if (!this.comments.length || isNaN(this.numSubreddits)) return;

            // Only run once per user
            if (this.newUser) {
                this.commentsSortedByScore = this.comments.slice(0).sort((a, b) => b.data.score - a.data.score);
                this.subredditCounts = this.calculateSubredditCounts();
            }

            let counts = this.subredditCounts.slice(0);

            if (counts.length > this.numSubreddits)
                counts.length = this.numSubreddits;

            return counts;
        },
        mostFrequentlyUsedWords() {
            if (this.isLoading || !this.comments.length || isNaN(this.numFrequentWords)) return;

            // Only run once per user
            if (this.newUser) {
                this.wordCounts = this.calculateWordCounts();
                this.newUser = false;
            }

            let counts = this.wordCounts.slice(0);

            if (counts.length > this.numFrequentWords)
                counts.length = this.numFrequentWords;

            return counts;
        },
        totalComments() {
            return this.comments.length;
        },
        totalSubmitted() {
            return this.submitted.length;
        },
        karmaPerComment() {
            if (!this.comments.length) return 'N/A';
            return Math.round(this.commentKarma/this.comments.length);
        },
        karmaPerSubmitted() {
            if (!this.submitted.length) return 'N/A';
            return Math.round(this.submittedKarma/this.submitted.length);
        },
        mostUpvotedComment() {
            if (!this.commentsSortedByScore) return;
            return this.commentsSortedByScore[0].data;
        },
        mostDownvotedComment() {
            if (!this.commentsSortedByScore) return;
            return this.commentsSortedByScore[this.commentsSortedByScore.length - 1].data;
        },
        kindnessScore() {
            let con = this.controversialityScore;
            let bwi = this.badWordIncidence;
            let nwi = this.niceWordIncidence;
            if (nwi === 0)
                nwi = .01;
            let negativeBehaviour = Math.round(5 * con * bwi/nwi); // 20% controversilaity is full strength
            if (negativeBehaviour > 100)
                negativeBehaviour = 100;
            return 100 - negativeBehaviour;
        },
        controversialityScore() {
            return Math.round(10 * this.calculateControversiality())/10;
        },
        userLink() {
            return `https://www.reddit.com/u/${this.username}`;
        },
        textComplexity() {
            const levels = ['very low', 'low', 'medium', 'high', 'very high', 'very high'];
            return levels[Math.floor(this.readingLevel / 20)];
        },
        readingLevelStyle() {
            return `width: ${this.readingLevel}%`;
        }
    },
    methods: {
        calculateSubredditCounts() {

            // Reset values for each user.
            this.commentKarma = 0;
            this.submittedKarma = 0;
            this.subreddits = {};

            let days = this.createSequenceOfDays(
                moment(1000 * this.comments[this.comments.length - 1].data.created_utc).format('YYYY-MM-DD'), // beginning (end of arr)
                moment(1000 * this.comments[0].data.created_utc).format('YYYY-MM-DD') // end (beginning of arr)
            );
            let daysWithComments = [];

            let arrIndex = 0;

            // Comments
            this.comments.slice(0).reverse().forEach(function(item) {
                daysWithComments.push(item);

                this.commentKarma += item.data.score;
                let day = moment(1000 * item.data.created_utc).format('YYYY-MM-DD');

                // Find the day in "days" array of objects then add the values.
                // There can be multiple items in the array with the same day.

                let newIndex = days.findIndex(item => item.day === day);

                if (newIndex === arrIndex) {
                    newIndex++;
                }
                arrIndex = newIndex;

                if (days[arrIndex]) {
                    days[arrIndex].numComments += 1;
                    days[arrIndex].commentKarma += item.data.score;
                    days[arrIndex].avgControversy = this.calculateControversiality(daysWithComments);
                }

                if (this.subreddits.hasOwnProperty(item.data.subreddit)) {
                    this.subreddits[item.data.subreddit] += 1;
                } else {
                    this.subreddits[item.data.subreddit] = 1;
                }
            }.bind(this));

            arrIndex = 0; // reset

            // Submissions
            this.submitted.slice(0).reverse().forEach(function(item) {

                let day = moment(1000 * item.data.created_utc).format('YYYY-MM-DD');

                let newIndex = days.findIndex(item => item.day === day);

                if (newIndex === arrIndex) {
                    newIndex++;
                }
                arrIndex = newIndex;

                if (days[arrIndex]) {
                    days[arrIndex].numSubmitted += 1;
                    days[arrIndex].submittedKarma += item.data.score;
                }

                this.submittedKarma += item.data.score;
                if (this.subreddits.hasOwnProperty(item.data.subreddit)) {
                    this.subreddits[item.data.subreddit] += 1;
                } else {
                    this.subreddits[item.data.subreddit] = 1;
                }
            }.bind(this));

            this.allDays = days.slice(0);
            this.smoothGraph();
            this.cumulative();

            let subredditCounts = [];
            for (let subreddit in this.subreddits) {
                subredditCounts.push(
                    {
                        name: subreddit,
                        count: this.subreddits[subreddit]
                    }
                )
            }
            subredditCounts.sort((a, b) =>  b.count - a.count);

            return subredditCounts;
        },
        calculateWordCounts() {

            let text = '';
            let frequencies = [];

            this.comments.forEach(function(item){
                text += item.data.body + ' ';
            });

            let words = text.split(/\W+/);
            const stopWords = 'a about above across actually after afterwards again against all almost alone along already also although always am among amongst amoungst amount amp an and another any anyhow anyone anything anyway anywhere are aren around as at back basically be became because become becomes becoming been before beforehand behind being below beside besides between beyond bill bit both bottom but by call can cannot cant co com con could couldnt cry cuz de describe detail did didn didnt do doesn doesnt doing done dont down due during each eg eight either eleven else elsewhere empty enough etc even ever every everyone everything everywhere except few fifteen fify fill find fire first five for former formerly forty found four from front full further get gif give go going good got gt had has hasnt have he hence her here hereafter hereby herein hereupon hers herself him himself his how however http https hundred ie if in inc indeed instead interest into is isn it its itself jpeg jpg just keep know last latter latterly least less let like look looks lot ltd ly made make makes many may maybe me meanwhile might mine more moreover most mostly move much must my myself name namely nbsp need neither never nevertheless next nine no nobody none noone nor not nothing now nowhere of off often oh ok okay on once one only onto or org other others otherwise our ours ourselves out over own part per perhaps please png pretty probably put rather re really said same see seem seemed seeming seems serious several she should shouldn shouldnt show side since sincere six sixty so some somehow someone something sometime sometimes somewhere still such sure take ten than thanks that thats the their them themselves then thence there thereafter thereby therefore therein thereupon these they think third this those though three through throughout thru thus to together too top toward towards trying twelve twenty two u un under until up upon us using very via want wanted was we well were weren werent what whatever when whence whenever where whereafter whereas whereby wherein whereupon wherever whether which while whither who whoever whole wouldn whom whose why will with within without would www ye yea yeah yet you your yours yourself yourselves';
            const badWords = ['fuck', 'fucking', 'fucked', 'ass', 'stupid', 'asshole', 'fucker', 'shit', 'shitty', 'bitch', 'dickhead', 'idiot',
                            'cunt', 'slut', 'sluts', 'whore', 'whores', 'nigger', 'niggers', 'terrible', 'awful', 'horrible', 'bad', 'prick',
                            'disgusting', 'gross', 'ugly', 'faggot', 'fag', 'fags', 'dumb', 'retard', 'retarded', 'retards', 'fat', 'cuck', 'cucks', 'cucked',
                            'brainwashed', 'bullshit', 'autist', 'autists', 'moron', 'morons', 'braindead', 'scum'];

            const niceWords = ['amazing', 'dope', 'lovely', 'sweet', 'thank', 'thanks', 'thankfully', 'appreciate', 'awesome', 'love', 'like', 'beautiful', 'gorgeous',
                            'accepting', 'adore', 'brilliant', 'happy', 'fantastic', 'generous', 'help', 'helpful', 'handsome', 'pretty', 'inspiring',
                            'cute', 'kindness', 'laugh', 'laughed', 'optimistic', 'perfect', 'peaceful', 'respect', 'smile', 'unreal', 'wholesome' ,'excellent',
                            'aww', 'friend', 'friendly', 'nice', 'haha', 'please', 'cool', 'interesting'];

            let saidABadWord = 0;
            let saidANiceWord = 0;
            let filteredWords = [];

            words.forEach(function(word){
                word = word.toLowerCase();
                if (isNaN(parseFloat(word)) && !isFinite(word)) {
                    if (!stopWords.includes(word)) {
                        if (frequencies.hasOwnProperty(word)) {
                            frequencies[word] += 1;
                        } else {
                            frequencies[word] = 1;
                        }
                        if (badWords.includes(word)) {
                            saidABadWord++;
                        }
                        if (niceWords.includes(word)) {
                            saidANiceWord++;
                        }
                    }
                    filteredWords.push(word);
                }
            }.bind(this));

            this.readingLevel = this.calculateTextComplexity(text, filteredWords);
            this.badWordIncidence = 100 * saidABadWord/words.length;
            this.niceWordIncidence = 100 * saidANiceWord/words.length;

            let wordCounts = [];
            for (let word in frequencies) {
                wordCounts.push(
                    {
                        name: word,
                        frequency: frequencies[word]
                    }
                );
            }
            wordCounts.sort((a, b) =>  b.frequency - a.frequency);

            return wordCounts;
        },
        calculateTextComplexity(text, words) {

            function numSyllables(word) {
                word = word.toLowerCase();
                if (word.length <= 2) return 1;
                word = word.replace(/(?:[^laeiouy]es|ed|[^laeiouy]e)$/, '');
                word = word.replace(/^y/, '');
                return word.match(/[aeiouy]{1,2}/g) ? word.match(/[aeiouy]{1,2}/g).length : 1;
            }

            let totalSyllables = 0;
            let totalSentences = 0;
            let sentenceRegex = new RegExp(/\w[.?!](\s|$)/g);
            while (sentenceRegex.exec(text) !== null) {
                totalSentences++;
            }
            words.forEach(function(word){
                totalSyllables += numSyllables(word);
            });
            return 100 - Math.round(
                    206.835 - ( 1.015 * (words.length / totalSentences) + 84.6 * (totalSyllables / words.length) )
                );
        },
        createSequenceOfDays(beginning, end) {
            let days = [];

            const beginningMinusOneDay = moment(1000 * moment(beginning).unix() - 86457 * 1000).format('YYYY-MM-DD');
            const endPlusOneDay = moment(1000 * moment(end).unix() + 86457 * 1000).format('YYYY-MM-DD');
            let unix = 1000 * moment(beginningMinusOneDay).unix();

            while (moment(unix).format('YYYY-MM-DD') !== endPlusOneDay) {
                days.push({
                    day: moment(unix).format('YYYY-MM-DD'),
                    numComments: 0,
                    commentKarma: 0,
                    numSubmitted: 0,
                    submittedKarma: 0
                });
                unix += 1000 * 86457; // milliseconds in a day (avg seconds in ~365.24 days)
            }

            return days;
        },
        calculateControversiality(comments = this.comments.slice(0)) {
            if (!comments.length) return 'N/A';
            if (comments.length < 5) return 0;
            let count = 0;

            comments.forEach(function(item){
                if (item.data.controversiality == 1) {
                    count += 1;
                }
            });
            return Math.round(100 * 100 * count / comments.length)/100;
        },
        subredditLink(subreddit) {
            return `https://www.reddit.com/r/${subreddit}/`;
        },
        googleLink(word) {
            return `https://www.google.com/search?q=${word}`;
        },
        percentageOf(val) {
            return Math.round(100 * val/(this.comments.length + this.submitted.length));
        },
        date(type, timestamp) {
            if (!timestamp) return;
            if (type === 'relative') {
                return moment(1000 * timestamp).fromNow();
            } else if (type === 'utc') {
                return moment(1000 * timestamp).format("MMM Do, YYYY");
            }
        },
        smoothGraph() {
            let numDays = this.allDays.length;
            if (numDays < 50)
                numDays = 50;
            const daysToRemove = Math.round(numDays/50);
            let counter = 0;

            let days = [];

            this.allDays.forEach(function(day){
                if (counter % daysToRemove === 0) {
                    days.push(day);
                }
                counter++;
            });

            this.days = days;
        },
        cumulative() {

            let cumulativeNumComments = 0;
            let cumulativeCommentKarma = 0;
            let cumulativeNumSubmitted = 0;
            let cumulativeSubmittedKarma = 0;

            this.allDays.forEach(function(day){
                cumulativeNumComments += day.numComments;
                cumulativeCommentKarma += day.commentKarma;
                cumulativeNumSubmitted += day.numSubmitted;
                cumulativeSubmittedKarma += day.submittedKarma;
                day.cumulativeNumComments = cumulativeNumComments;
                day.cumulativeCommentKarma = cumulativeCommentKarma;
                day.cumulativeNumSubmitted = cumulativeNumSubmitted;
                day.cumulativeSubmittedKarma = cumulativeSubmittedKarma;
            });

        }
    }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss">

$text: #cad6e6;

input[type=range] {
  -webkit-appearance: none;
  margin: 10px 0;
  width: 100%;
  background: transparent;

  &:active {
      &::-webkit-slider-thumb {
          opacity: 1;
          transition: transform 0.2s ease;
          transform: scale(1.5);
      }
      &::-moz-range-thumb {
          opacity: 1;
          transition: transform 0.2s ease;
          transform: scale(1.5);
      }
      &::-ms-thumb {
          opacity: 1;
          transition: transform 0.2s ease;
          transform: scale(1.5);
      }
  }
}
input[type=range]:focus {
  outline: none;
}
input[type=range]::-webkit-slider-runnable-track {
  width: 100%;
  height: 12.8px;
  cursor: pointer;
  animate: 0.2s;
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
  background: linear-gradient(135deg, #00bec7, #0073e8);
  border-radius: 25px;
  border: 0px solid #000101;
}
input[type=range]::-webkit-slider-thumb {
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
  border: 0px solid #000000;
  border-radius: 50%;
  height: 25px;
  width: 25px;
  background: white;
  cursor: pointer;
  -webkit-appearance: none;
  margin-top: -6px;
  opacity: 0.85;
}
input[type=range]:focus::-webkit-slider-runnable-track {
    background: linear-gradient(135deg, #00bec7, #0073e8);
}
input[type=range]::-moz-range-track {
  width: 100%;
  height: 12.8px;
  cursor: pointer;
  animate: 0.2s;
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
  background: linear-gradient(135deg, #00bec7, #0073e8);
  border-radius: 25px;
  border: 0px solid #000101;
}
input[type=range]::-moz-range-thumb {
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
  border: 0px solid #000000;
  border-radius: 50%;
  height: 25px;
  width: 25px;
  background: white;
  cursor: pointer;
  opacity: 0.85;
}
input[type=range]::-ms-track {
  width: 100%;
  height: 12.8px;
  cursor: pointer;
  animate: 0.2s;
  background: transparent;
  border-color: transparent;
  border-width: 39px 0;
  color: transparent;
}
input[type=range]::-ms-fill-lower {
    background: linear-gradient(135deg, #00bec7, #0073e8);
  border: 0px solid #000101;
  border-radius: 50px;
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
}
input[type=range]::-ms-fill-upper {
    background: linear-gradient(135deg, #00bec7, #0073e8);
  border: 0px solid #000101;
  border-radius: 50px;
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
}
input[type=range]::-ms-thumb {
  box-shadow: 0px 0px 0px #000000, 0px 0px 0px #0d0d0d;
  border: 0px solid #000000;
  border-radius: 50%;
  height: 25px;
  width: 25px;
  background: white;
  cursor: pointer;
  opacity: 0.85;
}
input[type=range]:focus::-ms-fill-lower {
    background: linear-gradient(135deg, #00bec7, #0073e8);

}
input[type=range]:focus::-ms-fill-upper {
    background: linear-gradient(135deg, #00bec7, #0073e8);
}

.input-rangeslider {
    max-width: 400px;
    margin: 0 auto;
    margin-bottom: 1rem;
}

.comment {
    word-wrap: break-word;
}

.container {
    &--summary {
        padding: 0;
    }
}

ul {
    list-style-type: none;
    padding: 0;
}

hr {
    border-top: 1px solid rgba(255,255,255,.1);
    margin: 2rem 0;
}

section.summary {
    padding: 4rem 0 2rem;
    margin-bottom: 4rem;
    border-radius: 10px;
    border-top-left-radius: 0;
    border-top-right-radius: 0;
    background-color: rgba(0, 0, 0, 0.2);
    border-top: 3px solid #7755ff;
}

h2 {
    font-family: 'Avenir', sans-serif;
    font-size: 2rem;
    font-weight: 300;
    text-align: center;
    margin-bottom: 2rem;
    font-weight: 500;
}
h3 {
    @extend h2;
    font-weight: 300;
    font-size: 1.75rem;
    margin-bottom: 1rem;
}

.controversiality {
    font-weight: 700;
    color: #ff5600;
}

p {
    font-size: 18px;
    color: $text !important;
}

a {
    transition: all 0.2s ease;
    &:hover {
        text-decoration: none;
        background-color: rgba(0,150,255,.1);
    }
}

.card {
    &--dark {
        cursor: pointer;
        background: rgba(0,0,0,0.5);
        box-shadow: 0 8px 40px -4px rgba(0, 0, 0, 0.5);
        margin: 0 0.5rem;
        padding: 0;
        border: none;
        margin-bottom: 1rem;
        border: 1px solid black;
        transition: all .2s ease;

        &:hover {
            background: linear-gradient(115deg, #ff79aa, #b743ff);
            box-shadow: 0 15px 100px -4px rgba(0, 0, 0, 0.7);
            color: white;
            transform: scale(1.1);
        }

        small {
            font-family: 'Avenir', sans-serif;
            opacity: 0.8;
        }

    }
}

ul {
    li {
        &:hover {
            a {
                color: white;
                background: transparent;
            }
        }
        &:first-child {
            font-weight: 600;
            box-shadow: 0 5px 60px -2px rgba(0, 50, 150, 0.25);
            background: #0073e8;
            background: linear-gradient(135deg, #00bec7, #0073e8);
            color: white;

            a {
                display: inline-block;
                color: white;
            }
        }
    }
    a {
        color: $text;
        padding: .4rem 0.8rem;
    }
}

p {
    text-align: center;
    color: $text;
    margin: 0 0 2rem;

    small {
        opacity: 0.8;
    }
}

.gradient-uppercase {
    font-size: 1.75rem;
    text-transform: uppercase;
    color: #96efff !important;
    background: -webkit-linear-gradient(115deg, #108aff, #96efff);
    -webkit-background-clip: text;
    -webkit-text-fill-color: transparent;
    border-radius: 30px;
    border: 1px solid #108aff;
    max-width: 300px;
    margin: 0 auto;
    margin-top: -0.5rem;
}

@media (min-width: 992px) {
    section.summary {
        padding: 5rem;
    }
}
</style>
