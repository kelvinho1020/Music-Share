<template>
	<div class='pb-10'>
		<!-- Music Header -->
		<section class="w-full mb-8 py-14 text-center text-white relative">
			<div class="absolute inset-0 w-full h-full box-border bg-contain" :class="{ 'background-dark': theme === 'dark', 'background-light': theme === 'light' }"></div>
			<div class="container mx-auto flex items-center w-full justify-between px-20">
				<!-- Play/Pause Button -->
				<div class="z-10 flex items-center w-full">
					<button type="button" class="z-50 h-24 w-24 text-3xl bg-white text-black rounded-full focus:outline-none" @click="newSong(song)">
						<i class="fas fa-play"></i>
					</button>
					<div class="z-50 text-left ml-8 w-3/4">
						<!-- Song Info -->
						<div class="text-3xl font-bold">{{ song.modified_name }}</div>
						<div class='break-all'>{{ song.description }}</div>
						<div>{{ song.createdAt }}</div>
					</div>
				</div>
				<button type="button" class="z-50 h-24 w-24 text-3xl focus:outline-none" @click="addFavorites" v-if="getUserLoggedIn">
					<i class="fa-heart" :class="{ fas: isFavorite, far: !isFavorite }"></i>
				</button>
			</div>
		</section>
		<!-- Form -->
		<section class="container mx-auto mt-6 px-20">
			<div class="bg-white rounded border border-gray-200 relative flex flex-col dark:bg-gray-600">
				<div class="px-6 pt-6 pb-5 font-bold border-b border-gray-200">
					<!-- Comment Count -->
					<span class="card-title dark:text-white">{{ comments.length }} comments</span>
					<i class="fa fa-comments float-right text-green-400 text-2xl"></i>
				</div>
				<div class="p-6">
					<div class="text-white text-center font-bold p-4 mb-10" v-if="submission" :class="alertClass">{{ alertMsg }}</div>
					<vee-form v-if="getUserLoggedIn" :validation-schema="schema" @submit="addComment">
						<vee-field
							as="textarea"
							name="comment"
							class="block w-full py-1.5 px-3 text-gray-800 border border-gray-300 transition duration-500 focus:outline-none focus:border-black rounded dark:bg-gray-700 dark:text-white dark:focus:border-gray-50"
							placeholder="Your comment here..."
						></vee-field>
						<ErrorMessage class="text-red-600 " name="comment" />
						<button type="submit" class="py-1.5 px-3 rounded text-white bg-green-600 block mt-4">Submit</button>
					</vee-form>
					<!-- Sort Comments -->
					<select class="block mt-4 py-1.5 px-3 text-gray-800 border border-gray-300 transition duration-500 focus:outline-none focus:border-black rounded focus:outline-none dark:focus:border-gray-50" v-model="sort">
						<option value="1">Latest</option>
						<option value="2">Oldest</option>
					</select>
				</div>
			</div>
		</section>
		<!-- Comments -->
		<ul class="container mx-auto mb-20 px-20 ">
			<li class="p-6 bg-white border border-gray-200 dark:bg-gray-600 dark:text-white w-full break-all" v-for="comment in sortedComments" :key="comment.docID">
				<!-- Comment Author -->
				<div class="mb-3">
					<div class="font-bold">{{ comment.name }}</div>
					<time>{{ comment.datePosted }}</time>
				</div>
				<p>{{ comment.content }}</p>
			</li>
		</ul>
	</div>
</template>

<script>
import { fromUnixTime, formatDistanceToNow, format } from "date-fns";
import { useStore } from "vuex";
import { songsCollection, commentsCollection, usersCollection, auth } from "@/includes/firebase";
import { useRoute, useRouter } from "vue-router";
import { computed, ref } from "vue";
export default {
	name: "song",
	setup() {
		// Router
		const route = useRoute();
		const router = useRouter();

		// Vuex
		const store = useStore();
		const getUserLoggedIn = computed(() => store.getters.getUserLoggedIn);

		const theme = computed(() => store.getters.getTheme);

		// FvoriteSong
		const favoriteSongs = ref([]);
		const isFavorite = computed(() => {
			if (favoriteSongs.value.indexOf(route.params.id) !== -1) {
				return true;
			} else {
				return false;
			}
		});

		const schema = {
			comment: "required|min:3|max:300",
		};

		const submission = ref(false);
		const showAlert = ref(false);
		const alertClass = ref("bg-blue-500");
		const alertMsg = ref("Please wait! Your comment is being submitted");

		// Song
		const song = ref([]);

		const newSong = function (song) {
			store.dispatch("newSong", song);
		};

		const getSong = async function () {
			const docSnapshot = await songsCollection.doc(route.params.id).get();

			if (!docSnapshot.exists) {
				router.push({ name: "Home" });
				return;
			}
			song.value = docSnapshot.data();
			song.value.createdAt = format(fromUnixTime(song.value.createdAt.seconds), "MM/dd/yyyy");

			document.title = `${song.value.modified_name} | Music-share`;
		};
		getSong();

		// comments
		const sort = ref("1");
		const comments = ref([]);
		const sortedComments = computed(() => {
			return comments.value
				.slice()
				.sort((a, b) => {
					if (sort.value === "1") {
						return new Date(b.datePosted) - new Date(a.datePosted);
					}

					return new Date(a.datePosted) - new Date(b.datePosted);
				})
				.map(comment => {
					return { ...comment, datePosted: formatDistanceToNow(new Date(comment.datePosted)) };
				});
		});

		const setComments = async function () {
			const snapshots = await commentsCollection.where("sid", "==", route.params.id).get();
			comments.value = [];
			snapshots.forEach(doc => {
				comments.value.push({
					docID: doc.id,
					...doc.data(),
				});
			});
		};
		setComments();

		const addComment = async function (values, context) {
			submission.value = true;
			showAlert.value = true;
			alertClass.value = "bg-blue-500";
			alertMsg.value = "Please wait! Your comment is being submitted";

			const comment = {
				content: values.comment,
				datePosted: new Date().toString(),
				sid: route.params.id,
				name: auth.currentUser.displayName,
				uid: auth.currentUser.uid,
			};

			await commentsCollection.add(comment);
			song.value.comment_count += 1;
			await songsCollection.doc(route.params.id).update({
				comment_count: song.value.comment_count,
			});

			setComments();
			submission.value = false;
			alertClass.value = "bg-green-500";
			alertMsg.value = "Comment added";

			context.resetForm();
		};

		// Favorite
		const setFavorites = async function () {
			const user = await usersCollection.doc(auth.currentUser.uid).get();
			if (user.data().favorite) {
				favoriteSongs.value = user.data().favorite;
			}
		};
		setFavorites();

		const addFavorites = async function () {
			try {
				if (favoriteSongs.value.indexOf(route.params.id) === -1) {
					favoriteSongs.value.push(route.params.id);
					await usersCollection.doc(auth.currentUser.uid).update({
						favorite: favoriteSongs.value,
					});

					await songsCollection.doc(route.params.id).update({
						favorite_count: +song.value.favorite_count + 1,
					});
				} else {
					favoriteSongs.value = favoriteSongs.value.filter(song => song !== route.params.id);
					await usersCollection.doc(auth.currentUser.uid).update({
						favorite: favoriteSongs.value,
					});
					await songsCollection.doc(route.params.id).update({
						favorite_count: +song.value.favorite_count - 1,
					});
				}
			} catch (err) {
				console.log(err);
			} finally {
				setFavorites();
			}
		};

		return { sort, getUserLoggedIn, schema, addComment, submission, showAlert, alertClass, alertMsg, newSong, song, comments, sortedComments, addFavorites, isFavorite, theme };
	},
};
</script>

<style scoped>
.background-light {
	background-image: url("../assets/img/user-header.png");
	animation: slide 50s linear infinite;
	will-change: background-position;
}
.background-dark {
	background-image: url("../assets/img/song-header.png");
	animation: slide 50s linear infinite;
	will-change: background-position;
}
@keyframes slide {
	0% {
		background-position: 0 0;
	}
	100% {
		background-position: -4000px 0;
	}
}
</style>
